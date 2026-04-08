# Lab 13: Jobs & CronJobs - Batch Task Execution

## Objective
Learn how to run one-off and recurring batch tasks in Kubernetes.

---

## What are Jobs and CronJobs?

**Job** - Creates one or more pods and ensures a specified number of them complete successfully.

**CronJob** - Schedules jobs to run periodically at specified times.

**Use cases:**
- Database migrations
- Batch processing
- Report generation
- Backup scripts
- Scheduled maintenance tasks

---

## Step 1: Create a Job

Create file `13-job-cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-job
  namespace: training
  labels:
    lesson: "13"
spec:
  # Number of successful completions needed
  completions: 1
  # Number of parallel pods
  parallelism: 1
  # Time limit for job
  activeDeadlineSeconds: 300
  # Retry limit for failed pods
  backoffLimit: 3
  template:
    metadata:
      labels:
        app: pi-job
    spec:
      containers:
        - name: pi
          image: perl:5.34
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
      # Job pods are not restarted automatically
      restartPolicy: Never
```

Apply:

```bash
kubectl apply -f 13-job-cronjob.yaml
```

---

## Step 2: Verify Job

```bash
# List jobs
kubectl get jobs -n training
kubectl get job -n training

# Describe job
kubectl describe job pi-job -n training
```

Expected: `Succeeded: 1`

---

## Step 3: Check Job Pod

```bash
# List pods created by job
kubectl get pods -n training -l app=pi-job

# Check job output
kubectl logs -n training -l app=pi-job
```

---

## Step 4: Create Parallel Job

Update job for parallel execution:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
  namespace: training
spec:
  completions: 5       # Need 5 successful completions
  parallelism: 2        # Run 2 pods in parallel
  template:
    metadata:
      labels:
        app: parallel-job
    spec:
      containers:
        - name: work
          image: busybox:1.36
          command: ["sh", "-c", "echo Processing item $ITEM && sleep 5 && echo Done"]
          env:
            - name: ITEM
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
      restartPolicy: Never
```

```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
  namespace: training
spec:
  completions: 5
  parallelism: 2
  template:
    metadata:
      labels:
        app: parallel-job
    spec:
      containers:
        - name: work
          image: busybox:1.36
          command: ["sh", "-c", "echo Processing item \$ITEM && sleep 5 && echo Done"]
          env:
            - name: ITEM
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
      restartPolicy: Never
EOF
```

---

## Step 5: Verify Parallel Execution

```bash
# Watch pods created
kubectl get pods -n training -l app=parallel-job -w

# Check job status
kubectl get job parallel-job -n training
```

---

## Step 6: Create CronJob

Add to file:

```yaml
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
  namespace: training
  labels:
    lesson: "13"
spec:
  # Schedule: minute hour day month weekday
  # "*/1 * * * *" = every minute
  schedule: "*/1 * * * *"
  # Successful jobs to keep
  successfulJobsHistoryLimit: 3
  # Failed jobs to keep
  failedJobsHistoryLimit: 1
  # Concurrency policy
  concurrencyPolicy: Forbid   # Forbid | Allow | Replace
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: hello-cron
        spec:
          containers:
            - name: hello
              image: busybox:1.36
              command: ["sh", "-c", "echo Hello from CronJob! && date"]
              resources:
                requests:
                  cpu: "10m"
                  memory: "16Mi"
                limits:
                  cpu: "50m"
                  memory: "32Mi"
          restartPolicy: OnFailure
```

Apply:

```bash
kubectl apply -f 13-job-cronjob.yaml
```

---

## Step 7: Verify CronJob

```bash
# List CronJobs
kubectl get cronjobs -n training
kubectl get cj -n training

# Describe CronJob
kubectl describe cronjob hello-cron -n training
```

---

## Step 8: Wait for CronJob Execution

```bash
# Wait for first execution (should happen within 1 minute)
kubectl get jobs -n training -l app=hello-cron --watch

# Check pods created
kubectl get pods -n training -l app=hello-cron

# Check output
kubectl logs -n training -l app=hello-cron
```

---

## Step 9: CronJob Schedule Formats

```bash
# Every minute
"*/1 * * * *"

# Every hour at minute 30
"30 * * * *"

# Every day at midnight
"0 0 * * *"

# Every Monday at 9am
"0 9 * * 1"

# Every 15 minutes
"*/15 * * * *"

# At 9:30am every weekday
"30 9 * * 1-5"
```

---

## Step 10: Concurrency Policies

```yaml
spec:
  # Allow - Allow concurrent jobs
  concurrencyPolicy: Allow

  # Forbid - Skip new job if previous still running
  concurrencyPolicy: Forbid

  # Replace - Cancel previous job, start new one
  concurrencyPolicy: Replace
```

---

## Step 11: Suspend CronJob

```bash
# Suspend execution
kubectl patch cronjob hello-cron -n training -p '{"spec":{"suspend":true}}'

# Check - no new jobs will run
kubectl get cronjob hello-cron -n training

# Resume
kubectl patch cronjob hello-cron -n training -p '{"spec":{"suspend":false}}'
```

---

## Step 12: Manual Job Execution

```bash
# Create job from CronJob manually
kubectl create job hello-cron-manual --from=cronjob/hello-cron -n training

# Or just run a one-off job
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: manual-job
  namespace: training
spec:
  template:
    spec:
      containers:
        - name: hello
          image: busybox:1.36
          command: ["echo", "Manual job execution"]
      restartPolicy: Never
EOF
```

---

## Step 13: Cleanup

```bash
# Delete jobs
kubectl delete job pi-job parallel-job -n training
kubectl delete job hello-cron-manual manual-job -n training

# Delete CronJob
kubectl delete cronjob hello-cron -n training

# List remaining (should be none)
kubectl get jobs -n training
kubectl get cronjobs -n training
```

---

## Verification Checklist

- [ ] Job created and completed successfully
- [ ] Job pods show correct output
- [ ] Parallel job runs multiple pods
- [ ] CronJob created and scheduled
- [ ] CronJob creates job at scheduled time
- [ ] Manual job execution works
- [ ] Suspend/resume works for CronJob
- [ ] Cleanup removes all resources

---

## Key Commands Summary

```bash
# Jobs
kubectl get jobs -n <namespace>
kubectl describe job <name> -n <namespace>
kubectl delete job <name> -n <namespace>

# CronJobs
kubectl get cronjobs -n <namespace>
kubectl get cj -n <namespace>
kubectl describe cronjob <name> -n <namespace>
kubectl delete cronjob <name> -n <namespace>

# Suspend/Resume
kubectl patch cronjob <name> -n <namespace> -p '{"spec":{"suspend":true}}'

# Manual execution
kubectl create job <name> --from=cronjob/<cronjob-name> -n <namespace>
```

---

## Troubleshooting

### Job not completing
```bash
# Check pod status
kubectl get pods -n <namespace> -l job-name=<job>

# Check pod logs
kubectl logs <pod> -n <namespace>

# Check events
kubectl describe job <job> -n <namespace>
```

### CronJob not creating jobs
```bash
# Check if suspended
kubectl get cronjob <name> -n <namespace> | grep SUSPEND

# Check schedule is valid
# Check successfulJobsHistoryLimit not 0
```

---

## Next Lab

Proceed to [Lab 14: Ingress](../k8s-training/14-ingress.yaml) - HTTP/HTTPS routing.
