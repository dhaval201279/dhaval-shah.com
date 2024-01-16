---
title: My journey to ace the CKAD exam
author: Dhaval Shah
type: post
date: 2024-01-12T06:00:50+00:00
url: /my-journey-to-ace-ckad-exam/
categories:
  - Kubernetes
tags:
  - kubernetes
  - certification
  - learning
thumbnail: "images/wp-content/uploads/2024/01/CKAD-certificate-image.png"
---


[![](https://www.dhaval-shah.com/images/wp-content/uploads/2024/01/CKAD-certificate-image.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/01/CKAD-certificate-image.png)
-----------------------------------------------------------------------------------------------------------------------------------------
# Background
As I concluded my year-end break in 2022, I engaged in a profound professional introspection. The realization that struck me most profoundly was the depth of my understanding of the intricate software systems I've been architecting, maintaining, and optimizing for numerous years. The stark truth was that I had merely scratched the surface of comprehending the infrastructure of the software products I've dedicatedly crafted. This realization prompted me to embark on the journey to earn my [CKAD](https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/) certification.

After dedicatedly preparing for 6-7 months, I successfully cleared the CKAD exam on 01Jan24, scoring 87%. This transformative journey not only enriched my knowledge of containers and Kubernetes but also exposed certain pitfalls that, if avoided, could have expedited my exam preparation.

In this article, I aim to share crucial insights, exam tips, and lessons learned from my CKAD journey, with the goal of aiding aspiring candidates in achieving success with greater ease.

**Disclaimer**
> I had no prior experience working on production-grade [Kubernetes](https://en.wikipedia.org/wiki/Kubernetes) clusters

# Understanding the CKAD exam
The CKAD exam is a practical, performance-based test spanning two hours, designed to immerse candidates in real-world Kubernetes tasks and challenges. Rather than squinting over multiple-choice questions, participants are tasked with creating, editing, deploying, and configuring various Kubernetes manifests in a remote cluster. It's an opportunity to experience hands-on learning and bid farewell to theoretical monotony. For more information about the exam, curriculum, and other essential details, visit the [CNCF's page](https://www.cncf.io/certification/ckad/)

# Navigating the Journey
As a novice to Kubernetes, I recognized the need to grasp the fundamentals of Kubernetes architecture and components. I began my journey with [Mumshad's Udemy Course](https://www.udemy.com/course/certified-kubernetes-application-developer/), tailored for individuals entirely new to the Kubernetes ecosystem. Consistently progressing through the videos and completing lab exercises on [kodecloud](https://kodekloud.com/courses/labs-certified-kubernetes-application-developer/) I gained a foundational understanding of the Kubernetes topics essential for CKAD success.

While tackling the 'Lightning Labs,' I identified gaps in my knowledge of various Kubernetes objects and components. This prompted me to explore additional resources utilized by fellow CKAD aspirants. The curated list below became instrumental in my preparation :
1. [Dimitris-Ilias Gkanatsios CKAD exercises](https://github.com/dgkanatsios/CKAD-exercises) - Providing an excellent set of problems and solutions for CKAD preparation.
2. [Benjamin Muschko's crash course](https://github.com/bmuschko/ckad-crash-course/tree/master) - Offering an in-depth and hands-on approach to practice problems covering CKAD exam topics
3. [Subod Dharma's CKAD preparation material](https://github.com/subodh-dharma/ckad/tree/master) - Featuring a comprehensive list of imperative commands and problem statements

Repeatedly utilizing these resources not only bolstered my confidence but also streamlined my problem-solving efficiency. They served as valuable reinforcements to the fundamentals acquired from Mumshad's course.

With enhanced confidence, I progressed to Mumshad's Udemy course lightning labs and mock exams.

# Strategies for CKAD Success
## Time Management: A Critical Factor
The CKAD exam's most challenging aspect is undoubtedly time management. With only two hours to complete 17-20 questions, which involve understanding problem statements, configuring key values, generating YAML files, and modifying existing Kubernetes objects, effective time utilization is paramount.

In order to be prepared from knowledge and overall examination standpoint, below are the few areas I was extremely mindful about -

## Mastering Imperative Commands
While YAML files are indispensable for managing Kubernetes infrastructure in real-world projects, relying solely on them in the exam can be time-consuming. Mastering imperative commands proves essential for efficiently addressing exam challenges. Hence,

> PLEASE AVOID WRITING YAML FILES FROM SCRATCH

Instead, use **_kubectl run_** and **_generators_** to improve productivity. I had extensively following imperative commands to create required K8 objects and generate yaml files -

### Pod
**Complete command**
``` bash
kubectl run nginx
  --image=nginx
  --restart=Never
  -l="key1=value1,key2=value2"  # multiple labels
  --env="ENV=prod" --env="region=US"  # multiple environment variables
  --port=5710  # container port
  --expose  # create a ClusterIP service
  --rm -it  # delete the pod after completed and open a terminal
  -- /bin/sh -c "while true; do date; sleep 10; done"  # command

```

**Generating yaml file**
``` bash
kubectl run my-pod --image=nginx --restart=Never -n default -l "key1=value1,key2=value2" --env="ENV=prod" --env="region=US" --port=80 $do -- /bin/sh -c ls > 4.yaml
```

**Modify pod's image**
``` bash
kubectl set image my-pod my-container=IMAGE_NAME:TAG
```

### Deployment
``` bash
kubectl create deploy my-deploy --image=nginx -n default --replicas=3 --port=8080 $do -- /bin/sh -c ls > 2.yaml
```

**Override image of deployment**
``` bash
kubectl set image deploy my-deployment my-container-name=nginx:1.9.1 --record # save the "CHANGE-CAUSE" in the deployment history
```

**Managing deployment rollouts**
``` bash
kubectl rollout history my-deploy                                # Check the history of deployments including exact revision with --revision=2
kubectl rollout undo my-deploy                                   # Rollback to the previous deployment
kubectl rollout undo my-deploy --to-revision=2                   # Rollback to a specific revision. If not specified, rollback to previous version
kubectl rollout status -w my-deploy                              # Watch rolling update status of "frontend" deployment until completion
kubectl rollout restart my-deploy 
```

### Service
**Cluster Service**
``` bash
kubectl create svc clusterip my-cluster-svc --tcp=6379:6379  # <port>:<targetPort>
```

**Node Port Service**
``` bash
kubectl create svc nodeport my-nodeport-svc --tcp=80:80 --node-port=30080  # <port>:<targetPort>
```

**Exposing existing deployment / pod as a service**
``` bash
kubectl expose deploy my-deployment --port=80 --target-port=8080 --type=ClusterIP --name=my-deploy-svc $do > 5.yaml
```

**Note** - Both the above commands i.e. direct service creation and creating service from existing deployment / pod have their own limitations - Direct service creation cannot accept selector and the other cannot accept a node port.

### Job
``` bash
kubectl create job my-job --image busybox -n default $do -- /bin/sh -c ls > 1.yaml
```

### Cron Job
``` bash
kubectl create cj my-cronjob --image busybox --schedule "*/1 * * * *" -n default --restart Never $do -- /bin/sh -c ls > 1.yaml
```
**Creating job from cron job**
``` bash
kubectl create job my-job --from=cronjob/my-cronjob
```

### Config Maps / Secrets
``` bash
kubectl create cm app-config
  --from-literal=key123=value123

kubectl create cm app-config-env-file
  --from-env-file=config.env  # single file with environment variables

kubectl create cm app-config-file
  --from-file=config.txt  # single file or directory
```
**Note** - For creating secret replace ***cm*** with ***secret generic***

### Service Account
``` bash
kubectl create sa my-sa
```

### Resource Quota
``` bash
kubectl create quota myrq --hard=cpu=1,memory=1G,pods=2
```

### Labelling / Annotating objects
``` bash
kubectl label nodes my-node env=PROD            # add a label to node
kubectl label pod my-pod region=eu              # add a label to pod
kubectl label pod my-pod region=us --overwrite  # edit a label
kubectl label pod my-pod region-                # remove a label
```

**Note** - For annotation, replace ***label*** with ***annotate***

### Fetching pods based on labels
``` bash
kubectl get po
  -l env=prod,reqion!=US     # label equality filter
  -l 'env in (prod, stage)'  # label set filter
  -l 'region notin (US, EU)' # notin filter
  -l 'env exists'            # exists filter
  --show-labels              # show all labels
```

### Horizontal Pod Autoscaler (HPA)
``` bash
kubectl autoscale deploy my-deploy
        --cpu-percent=70    # sum of the cpu percentages used by all the pods that needs to be reached for the hpa to start spinning other pods
        --min=2             # min number of pods
        --max=8             # max number of pods the HPA can scale
```
**Note** - In order to enable HPA with memory related critera, we will have to create / update yaml file. Refer [HPA docs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

### Taint
``` bash
# apply taint
kubectl taint nodes my-node my-key=my-value:NoSchedule

# remove taint
kubectl taint nodes my-node my-key=my-value:NoSchedule-
```

### Debugging in Kubernetes
**Get pod logs**
``` bash
kubectl logs my-pod

kubectl logos my-pod -p     #Fetch logs of previous instance

kubectl logs my-pod -c my-container -f #Fetch logs of specific container
```

**Open pod shell**
``` bash
kubectl exec -it my-pod -- /bin/sh
```

**View Service endpoints**
``` bash
kubectl get ep
```

**Verifying K8 objects with its corresponding apiVersion**
``` bash
kubectl api-resources | grep deployment
```

**Fetch all events**
``` bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

**View kubernetes API documentation**
``` bash
kubectl explain pods.spec
kubectl explain pod.spec > po-spec
kubectl explain pod –recursive=true > po-full
```
## Non-Kubernetes Proficiency
One distinctive aspect of the CKAD exam is, its assessment is not only of Kubernetes expertise but also of proficiency in [Linux](https://en.wikipedia.org/wiki/Linux) skills with a focus on -

1. Vim editor
2. Grep
3. Shortcuts with alias and export variables

### Vim editor
I had used [Vim editor](https://en.wikipedia.org/wiki/Vim_(text_editor)) extensively for creating and editing YAML files essential in the management of Kubernetes objects. A thorough command of Vim shortcuts for seamless navigation and editing of YAML files is imperative.

#### Vim editor configurations
To enhance the effectiveness and productivity of the Vim editor, it is recommended to tailor configurations in the _vimrc_ file as shown below

``` bash
set nu    # S number line
set et    # Use spaces for tab
set ts=2  # Amount of spaces used for tab
set sw=2  # Amount of spaces used during indentation
set ai    # Sets auto indent
set hls   # Highlights the term searched in editor
set ic    # Ignores case while searching
set cuc   # Enables cursor on column
set cul   # Enables cursor on line
syntax on
```

#### Vim editor navigation shortcuts
As time is limited for CKAD exam, one needs to be as optimal as possible to save time. Hence it is recommended to remember these shortcuts for faster navigation -

``` vim
w / W -> Beginning of next word
e / E - End of current or next word (next word in case it is already at the end of current word)
0 - Beginning of current line
$ - End of current line
d$ - Delete from current cursor position to end of line
:/<search-string> - Search entered search string
  a. n - Next occurrence
  b. N - Previous occurrence
u - Undo last action
U - Undo all changes to current line
Copy / Cut - Paste
  a. Press ESC
  b. Take the cursor from the lines you want to copy / delete from
  c. 5yy / 5dd - Copy / Cut 5 lines from current line
  d. Take the cursor to the location where you want to paste copied contents
    i. p - Paste copied / cut lines below the line where cursor is placed
    ii. P - Paste copied / cut lines above the line where cursor is placed
Multiline indent
  a. Switch to visual mode
  b. Move to the top of code block that need to be indented. Select single line with 'Shift + V", and than use arrow to select multiple lines
  c. Selected lines will show as highlighted
  d. Use "Shift + >" (Greater than sign and not an arrow key) / "Shift + <>" to indent towards right / left. The number of spaces of indentation is determined by value of set ts=2 in .vimrc file

```

### Grep command
Extensive use of the grep command is required for cluster validation and searching Kubernetes documentation via the command line. This demands a strong command over this powerful text-searching tool. I had used below flags extensively for effective searching - 
1. ***-i*** - Ignore case while searching
2. ***-A*** - Prints n lines after the match
3. ***-B*** - Prints n lines before the match

### Shortcuts with alias and export
Proficiency in utilizing shortcuts, creating aliases, and managing exported variables is crucial for efficient and swift command-line operations. Modify ***.bashrc*** file with below aliases and exported variables to expedite formulation of K8 commands.

``` bash
alias kns="kubectl config set-context --current --namespace"
alias gns="kubectl config view | grep -i namespace"
alias ka="kubectl apply -f"
alias ksh="kubectl exec -it"
alias ktn="kubectl run tmp1 --image nginx:alpine --restart Never -it --rm -- /bin/sh"
alias ktb="kubectl run tmp2 --image busybox --restart Never -it --rm -- /bin/sh"
alias cls=clear

export do="--dry-run=client -o yaml"
export fg="--force --grace-period=0"
export sl="--show-labels"
```

### Navigating the exam environment
Another crucial factor that significantly contributes to efficiency during the CKAD exam is familiarity with the exam environment. Utilize [killer.sh](https://killercoda.com/killer-shell-ckad) judiciously, as you are granted two opportunities to use it before the exam, each lasting for a fixed duration of 36 hours. killer.sh will mimic actual environment along with the exact set of tools that would be available during exam. Few things that I had consciously taken care of -

#### Copy / Paste in PSI browser
The default configuration for 'Copy/Paste' within the remote box in the PSI browser incessantly prompts an 'Unsafe paste dialog.' To disable this feature, follow these steps:
1. Go to ***Applications*** -> ***Settings*** and click ***Xfce Terminal***
2. Uncheck ***Show unsafe paste dialog*** and check ***Automatically copy selection to clipboard***

#### Using Mozilla browser
Given that the PSI browser's remote box is equipped with the Mozilla browser, it's essential to grasp its optimal utilization for navigation and searching. The browser primarily serves for accessing K8 Documentation. I found the following shortcuts particularly helpful in enhancing search speed within the Mozilla browser:

1. ***Ctrl + F*** - Find text within rendered web page
2. ***/*** - Quick search bar
3. ***highlight all*** - Enable this option within search bar to highlight searched items within rendered web page
4. ***Search next item*** - _F3_ or _Ctrl + G_
5. ***Search previous item*** - _Shift F3_ or _Ctrl + Shift + G_

#### Mousepad
Consider Mousepad as your digital notepad, where you may wish to store essential texts or notes for quick reference.

## Conclusion
In the pursuit of achieving success in the CKAD exam, a holistic approach encompassing both Kubernetes proficiency and non-Kubernetes skills is paramount. The journey involves not only mastering the intricacies of Kubernetes architecture but also honing essential Linux skills. Proficiency in tools such as the Vim editor, understanding the nuances of Grep, and adept usage of shortcuts, aliases, and export variables are indispensable.

Success in the CKAD exam is not solely measured by Kubernetes knowledge but also by one's ability to navigate the practical challenges presented in the exam environment. By adopting a comprehensive strategy that encompasses both technical skills and strategic exam management, candidates can confidently approach the CKAD certification journey and emerge victorious.