Pipeline based on documentation here: https://docs.openshift.com/container-platform/4.13/cicd/pipelines/creating-applications-with-cicd-pipelines.html

Create secret with Azure Personal Access Token (only Repo Read permissions needed): 


```console
cat <<EOF | oc create -f - 
apiVersion: v1
stringData:
  password: MY_PERSONAL_ACCESS_TOKEN
  username: git  # can be whatever you want
kind: Secret
metadata:
  annotations:
    tekton.dev/git-0: https://dev.azure.com
  name: git-creds  
type: kubernetes.io/basic-auth
EOF
```

Link the secret to the `pipeline` service account. 

```console
oc secrets link pipeline git-creds
```

Create ConfigMap to be mounted and used by the polling CronJob. Every line contains an Azure DevOps git repository whcih in the previous step given PAT has granted access to, an event listener name and the enumeration of branches, whose changes will trigger the pipeline. Repository, Event listener and branches will be separated by '|' and branches by ';'.  For instance

```
REPOS_CM_FILE=/tmp/repos
cat <<EOF > $REPOS_CM_FILE
https://dev.azure.com/MY_AZURE_DEVOPS_ORGANIZATION/MY_AZURE_DEVOPS_PROJECT/_git/MY_AZURE_DEVOPS_REPOSITORY_1|my-event-listener-for-pipeline-1:8000|main;develop
https://dev.azure.com/MY_AZURE_DEVOPS_ORGANIZATION/MY_AZURE_DEVOPS_PROJECT/_git/MY_AZURE_DEVOPS_REPOSITORY_2|my-event-listener-for-pipeline-1:8000|main;qa
https://dev.azure.com/MY_AZURE_DEVOPS_ORGANIZATION/MY_AZURE_DEVOPS_PROJECT/_git/MY_AZURE_DEVOPS_REPOSITORY_3|my-event-listener-for-pipeline-2:8000|main
EOF
oc create configmap input-git --from-file=$REPOS_CM_FILE
```

Create the CronJob for polling (executed every minute) 

```console
cat <<EOF | oc create -f -
kind: CronJob
apiVersion: batch/v1
metadata:
  name: polling-triggers  
spec:
  schedule: '*/1 * * * *'
  concurrencyPolicy: Allow
  suspend: false
  jobTemplate:    
    spec:
      template:        
        spec:
          volumes:
            - name: input-git
              configMap:
                name: input-git
                defaultMode: 420
            - name: repolist
              persistentVolumeClaim:
                claimName: repolist-pvc
          containers:
            - resources: {}
              terminationMessagePath: /dev/termination-log
              name: polling-triggers
              env:
                - name: AZURE_DEVOPS_PAT
                  valueFrom:
                    secretKeyRef:
                      name: git-creds
                      key: password
                - name: BASEDIR
                  value: /repolist
                - name: JSONTEMPLATE
                  value: >-
                    {"object_kind": "push","event_name": "push","head_commit":
                    {"id": "GITHUBREPOREV"},"repository": {"name":
                    "GITHUBREPONAME","url": "GITHUBREPOURL"}}
              imagePullPolicy: Always
              volumeMounts:
                - name: repolist
                  mountPath: /repolist
                - name: input-git
                  mountPath: /info/repos.txt
                  subPath: repos.txt
              terminationMessagePolicy: File
              image: 'registry.redhat.io/rhel8/toolbox:latest'
              args:
                - /bin/sh
                - '-c'
                - |
                  set -eu 
                  input="/info/repos.txt"
                  i=1
                  at=@
                  while read line || [ -n "$line" ]; do
                    echo "$i: $line"
                    position=1
                    while IFS='|' read -ra ADDR; do
                      for item in "${ADDR[@]}"; do            
                        case $position in
                          1)
                            echo "  - Repository URL: $item"          
                            REPO_BASE=${item#*$at}
                            REPO_BASE_W_TOKEN="https://git:$(echo $AZURE_DEVOPS_PAT | tr -d '\n')@${REPO_BASE}"
                            REPO_NAME=${item##*/}
                            echo "      Base URL    : $REPO_BASE"
                            echo "      Repository  : $REPO_NAME"                                                        
                            ;;
                          2)
                            echo "  - Event Listener: $item"
                            EVENT_LISTENER=$item
                            ;;
                          3)
                            echo "  - Branches      : $item"
                            while IFS=';' read -ra BRANCH; do
                              for REPOBRANCH in "${BRANCH[@]}"; do
                                echo "     branch: $REPOBRANCH"                              
                                # revision initialization
                                echo  " _current_revision=$(git ls-remote --heads ${REPO_BASE_W_TOKEN} ${REPOBRANCH} | awk '{print $1}')"

                                _current_revision=$(git ls-remote --heads ${REPO_BASE_W_TOKEN} ${REPOBRANCH} | awk '{print $1}')
                                _prev_revision=${_current_revision}

                                # replace slash chars (/) by underscores (_) in branches (feature/my_feature will be feature_my_feature)
                                REPO_SHA256_FILE=${BASEDIR}/${REPO_NAME}_${REPOBRANCH//\//_}.sha256
                                # if there is not existing previous revision data file, creating new revision data file with current revision
                                test -f ${REPO_SHA256_FILE} && _prev_revision=$(cat ${REPO_SHA256_FILE}) || echo ${_current_revision} > ${REPO_SHA256_FILE}

                                # generating JSON data
                                _jsondata=$(echo ${JSONTEMPLATE} | sed -e "s=GITHUBREPOREV=${_current_revision}=" -e "s=GITHUBREPONAME=${REPO_NAME}=" -e "s=GITHUBREPOURL=https://${REPO_BASE}=")

                                # check if there are any changes through comparing previous and current revisions.
                                # If there are any changes, trigger a new pipeline using curl and json data.
                                test "${_current_revision}" != "${_prev_revision}" && echo ${_current_revision} > ${REPO_SHA256_FILE} &&
                                curl -s -X POST -H 'Content-Type: application/json' -H 'X-GitHub-Event: push' \
                                     -d "${_jsondata}" ${EVENT_LISTENER} ||
                                echo "No changes" 
                              done
                            done <<< $item                          
                            ;;
                          *)
                            echo "ERROR:  Error detected processing line $i"
                            ;;
                        esac
                        ((position=position+1))
                      done    
                    done <<< "$line"
                    ((i=i+1))
                  done < "$input"
          restartPolicy: OnFailure
          terminationGracePeriodSeconds: 30
          dnsPolicy: ClusterFirst
          securityContext: {}
          schedulerName: default-scheduler
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```
