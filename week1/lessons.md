# Week 1: Deployment Lessons Learned

## Overview
Key technical challenges and solutions encountered while deploying a Next.js application to AWS Lambda using Docker and ECR.

---

## Lesson 1: Environment Variable Persistence in Docker Builds

### Problem
Docker image was being pushed to `ap-south-2` region instead of the intended `ap-south-1`, despite `.env` file containing the correct region.

### Root Cause
Environment variables were cached from a previous session and not reloaded during the build process.

### Solution
Explicitly reload environment variables before Docker build and push operations.

### Key Takeaway
**Environment variables are not automatically refreshed between shell sessions.** Always verify active environment state before deployment operations, especially when switching regions or configurations.

### Interview Talking Points
- Demonstrates understanding of Docker build-time vs runtime context
- Shows systematic debugging approach: checked config files first, then runtime state
- Highlights importance of environment isolation in deployment pipelines

---

## Lesson 2: AWS Lambda Image Format Compatibility

### Problem
Lambda function creation failed with error:
```
The image manifest, config or layer media type for the source image 
122610525143.dkr.ecr.ap-south-1.amazonaws.com/consultation-app@sha256:... 
is not supported.
```

### Root Cause
Modern Docker BuildKit creates images in **OCI format** with provenance attestations by default. AWS Lambda only supports **Docker v2 Schema 2 format**.

### Technical Details
- Docker BuildKit adds attestation metadata that creates an image index
- This image index structure is incompatible with Lambda's image processing
- Lambda requires the simpler Docker v2 Schema 2 manifest format

### Solution
Add `--provenance=false` flag to Docker build command:
```bash
docker build --provenance=false -t <image-name> .
```

### Key Takeaway
**Service compatibility requires understanding format specifications.** Modern tools may use newer standards that aren't universally supported across cloud services.

### Interview Talking Points
- Shows awareness of image format standards (OCI vs Docker v2)
- Demonstrates ability to bridge compatibility gaps between tools and platforms
- Highlights importance of reading error messages carefully (manifest/media type was the key hint)
- Reflects understanding of AWS Lambda's container image requirements

---

## Lesson 3: Strategic Service Selection - App Runner Deprecation Response

### Problem
Initially deployed the application using AWS App Runner. On April 30, 2026, AWS stopped accepting new customers for App Runner and recommended migrating to **AWS ECS Express Mode** as the official replacement.

### Business Context
ECS Express Mode, while being the official AWS-recommended path, introduces significant architectural and cost implications:
- **Automatically provisions an Application Load Balancer (ALB)**: ~$16-20/month baseline cost
- **ALB costs persist even when application is idle**: no scale-to-zero capability
- **Additional infrastructure overhead**: security groups, target groups, VPC configuration, ACM certificates
- **Complex cleanup**: not all resources auto-delete when service is removed

### Decision Criteria
For a learning/demo project with intermittent traffic:
1. **Cost efficiency**: need scale-to-zero to avoid idle charges
2. **Simplicity**: minimize infrastructure components
3. **Equivalent functionality**: maintain HTTPS endpoint, container support, streaming responses
4. **Production-ready**: solution must be viable beyond proof-of-concept

### Solution Architecture
Implemented **AWS Lambda with Container Image support** instead of ECS Express Mode:

**Core Components:**
- **AWS Lambda**: Serverless compute running Docker container (up to 10GB)
- **Lambda Function URLs**: Direct HTTPS endpoint without API Gateway
- **AWS Lambda Web Adapter**: Extension enabling standard web frameworks (FastAPI) to run on Lambda unchanged

**Why This Works:**
- **Scale to zero**: Lambda only runs during invocations; $0 cost when idle
- **No load balancer required**: Function URLs provide direct HTTPS access
- **Same container image**: No application code changes; same Dockerfile works
- **Response streaming support**: Function URL with `RESPONSE_STREAM` mode enables Server-Sent Events
- **Cost**: Stays within Lambda free tier (1M requests/month, 400k GB-seconds/month perpetual free tier)

### Implementation Details

**Lambda Web Adapter Integration** (only 3 lines added to Dockerfile):
```dockerfile
# Add Lambda extension (inert in local Docker; activates on Lambda)
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:1.0.0 /lambda-adapter /opt/extensions/lambda-adapter
ENV PORT=8000
ENV AWS_LWA_INVOKE_MODE=response_stream
```

**Configuration:**
- Memory: 1024 MB (comfortable headroom for FastAPI)
- Timeout: 5 minutes (accommodates longest streaming responses)
- Reserved concurrency: 2 (cost protection cap)
- Function URL invoke mode: `RESPONSE_STREAM` (critical for SSE)

### Key Takeaway
**Official recommendations aren't always optimal for your use case.** Evaluated AWS's guidance against actual requirements and chose a simpler, cheaper, equally production-ready alternative. Understanding service pricing models and architectural constraints is as important as technical implementation.

### Cost Comparison

| Solution | Monthly Cost (Idle) | Monthly Cost (Light Usage) | Complexity |
|----------|---------------------|----------------------------|------------|
| App Runner | ~$5-6 | ~$5-10 | Low |
| ECS Express Mode | ~$16-20 (ALB alone) | ~$20-30 | High |
| **Lambda + Function URL** | **$0** | **~$0.10** | **Low** |

### Interview Talking Points
- **Cost-aware architecture**: Recognized that official migration path wasn't cost-optimal for the use case
- **Service evaluation skills**: Compared Lambda containers vs ECS based on actual requirements (scale-to-zero, simplicity, cost)
- **AWS service breadth knowledge**: Knew Lambda supports containers (added 2020) and Function URLs support streaming (added 2025)
- **Pragmatic decision-making**: Chose production-ready solution that stays free vs. "recommended" solution costing $200+/year
- **Container portability**: Same Docker image works across Lambda, ECS, App Runner, or any container runtime
- **Extension ecosystem awareness**: Leveraged AWS Labs' Lambda Web Adapter to bridge standard web frameworks to Lambda's event model

### Production Readiness
This isn't just a cost-cutting hack—Lambda containers are production-grade:
- Used by companies serving billions of requests
- Supports all standard observability (CloudWatch Logs, X-Ray, metrics)
- Integrates with CI/CD pipelines
- Scales automatically from zero to thousands of concurrent executions
- Same container can be moved to ECS/Fargate if requirements change (e.g., need 15+ minute execution times)

### Technical Depth Demonstrated
- **AWS service evolution awareness**: App Runner deprecation, Lambda container images (2020), Function URL streaming (2025)
- **Cost model understanding**: ALB is billed per hour + per LCU; Lambda is pure pay-per-use
- **Format compatibility**: Lambda requires Docker v2 Schema 2 (see Lesson 2)
- **Streaming semantics**: Configured response streaming at both Lambda (invoke mode) and application (FastAPI SSE) layers

---

## Architecture Principles Applied

1. **Configuration Management**: Verified environment state before deployment
2. **Compatibility Awareness**: Understood service constraints and adjusted tooling accordingly
3. **Root Cause Analysis**: Looked beyond surface errors to understand underlying technical issues
4. **Cost Optimization**: Evaluated official recommendations against actual cost/complexity tradeoffs
5. **Service Selection**: Chose appropriate AWS services based on workload characteristics
6. **Documentation**: Captured solutions for repeatable deployments

---

## Production Readiness Considerations

- **CI/CD Integration**: These flags and Lambda update commands should be codified in build scripts and CI/CD pipelines
- **Multi-Region Deployments**: Environment validation becomes critical when deploying across regions
- **Image Format Standards**: Document format requirements in deployment runbooks
- **Cost Monitoring**: Lambda free tier is generous but set up budgets and alarms anyway
- **Service Deprecation Awareness**: Monitor AWS announcements; have migration plans ready
- **Debugging Strategy**: Systematic approach saved time vs trial-and-error

---

## Week 1 Summary

Successfully deployed a production-grade containerized SaaS application to AWS Lambda, navigating:
- Environment configuration issues
- Container image format compatibility
- AWS service deprecation and strategic re-architecture

**Total deployment cost**: $0/month (within Lambda free tier)  
**Deployment time**: ~30 minutes after solving initial issues  
**Architecture**: Docker container → ECR → Lambda → Function URL → public HTTPS endpoint

---

*Document created: Week 1 completion*  
*Target: Principal Forward Deployed Engineer / Principal Solutions Architect interviews*
