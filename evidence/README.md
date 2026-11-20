# Evidence

| File | Shows |
|---|---|
| `cloud-architecture-lab.png` | Full design built in the Cloud Architecture Lab (2 AZs, 6 subnets, 46 components) |
| `lab-02-w1-open-exam-autosave.png` | Walkthrough 1 simulated: autosave → DynamoDB, marking job → SQS, HTTP 202 in 147 ms |
| `lab-03-w2-upload-notify.png` | Walkthrough 2 simulated: 40 MB upload → S3 → SQS → Lambda notify tutor, HTTP 202 |
| `lab-04-exam-peak-100x.png` | Walkthrough 1 at 100× spike load, HTTP 202 |
| `lab-05-az-a-failure.png` | Failure Lab: AZ-A outage, ALB routes to AZ-B, HTTP 202 |
| `lab-06-az-b-failure.png` | Failure Lab: AZ-B outage, ALB routes to AZ-A, HTTP 202 |
| `01-project-structure.png` | Repository layout |
| `05-cfn-lint.png` | CloudFormation template passes cfn-lint |
| `06-python-checks.png` | API code passes py_compile + pyflakes |
| `07-git-commit.png` | Commit history |
| `08-git-push.png` | Remote and push status |

The lab workspace itself is [`../architecture/lab-architecture.json`](../architecture/lab-architecture.json). To reopen it, go to Cloud Architecture Lab → Export → Upload JSON. Results are explained in section 8 of [`../report.md`](../report.md).
