TranZact Technical Test

Name : Naman 
Roll no: 23BEC073

## Warm Greetings, Mr.sun337 and Mr.rss-holmes.##

-----------------------------------------------Design A PDF genration microservice-------------------------------------------------------

## Initial Understanding :
The core objective is to build an independent PDF generation microservice that scales for sudden traffic spikes without requiring a re-architecture of the primary monolithic Django application.
The primary difficulty lies in balancing extreme resource demands with a strict financial constraint. Puppeteer is highly memory-intensive, consuming ~400MB per render. Attempting to instantaneously process the peak load of up to 1000 single requests and 10 bulk requests (up to 100 PDFs each) per minute will immediately cause Out of Memory (OOM) crashes, especially when parsing dynamic tables with 500+ rows. Designing a system that gracefully throttles this massive RAM requirement while keeping total monthly infrastructure costs under ₹12,500 (~$150 USD) is the central architectural challenge.

## Assumptions :
The monolithic Django application can transmit the complete, finalized data state for all required invoices within the initial JSON payload.
AWS S3 (or an equivalent cloud object storage) is permitted within the architecture for file storage and archiving.

## Clarification Question : 
What is the acceptable time limit for a bulk generation to be considered "completed within a reasonable time ?

## Capacity Planning & Math :

Processing Load : 
Peak Single Load: ~1000 requests/minute. (Given)
Peak Bulk Load: 10 requests * 100 PDFs = 1000 requests/minute. (Given)
Total Peak Load(in min) : 2000 PDFs/minute
Total Peak Load(in sec) : 33.33 PDFs/second

## Memory Requirements : 
Each PDF takes ~3 seconds to render. (Given)
For instant processing: 33.3 PDFs/sec * 3 seconds = 100 concurrent workers.
Total Memory: 100 workers * 400MB = 40,000MB (~40GB RAM).

We need to use a 64GB RAM here which costs $320 per month (Researched). Completely over our budget.
This means instantaneous processing of bulk is impossible with our current budget. We need to asyncronously process the bulk requests.

## Design Decisions : 

1. We are on budget constraints with our processing load, we need to trade of bulk request/min.

How can we solve this :

Peak load: ~1,000 single requests/minute.

Total volume: ~30,000 PDFs/month. (This is the reason we use this approach) (Given)

If you divide 30,000 PDFs by 30 days, that is only 1,000 PDFs per day.

This means your system is almost entirely idle 99% of the time, and then suddenly gets hit with a massive spike of traffic all at once. If you rent a 20GB or 32GB server and run it 24/7 for an entire month just to handle a few minutes of peak traffic, you are burning money.

The Approach we used : 
EC2 Auto-Scaling Group (If you must use standard servers) :
How it works: You run one tiny, cheap server (like a 2GB instance for $15/month) to handle the idle times. You connect it to an AWS CloudWatch alarm. When the Redis queue suddenly fills up with 1,000 requests, the alarm triggers and AWS automatically launches three or four 8GB servers to churn through the queue.

The Cost: Once the queue is empty, the extra servers automatically delete themselves. You only pay for the few minutes the big servers were alive.

Alternatives : 
Serverless Architecture (AWS Lambda)
Instead of renting an always-on EC2 server, you can build the PDF generation microservice using AWS Lambda.

How it works: Lambda only spins up and charges you when a request arrives. If 1,000 requests arrive at the same second, AWS instantly spins up 1,000 separate, isolated micro-containers.

The Cost: You pay per millisecond of compute time. 30,000 PDFs * 3 seconds per generation = 90,000 seconds of compute time per month. On AWS Lambda, this costs less than $5 per month, easily beating the $150 budget while handling the 1000/min peak effortlessly.

The Trade-off: "Cold starts." The very first time a Lambda function spins up, it might take an extra 1-2 seconds to initialize Puppeteer.


2. Isolating OOM Crashes :
The dynamic tables with 500+ rows cause Puppeteer to crash mid-render with Out of Memory (OOM) errors. (Given)

Approach we used :
Visibility Timeout / Dead Letter Queue (DLQ): 
When a worker pulls a job from Redis, it is temporarily hidden from other workers. If the worker crashes (OOM) and fails to confirm completion within a set time, the job reappears in the queue.

Max Retries: If a specific job causes 3 crashes in a row, the system moves it to a "Dead Letter Queue" (a separate list for failed tasks) and flags the batch as "Failed" so the system does not get stuck in an infinite crash loop.

3. Handling Bulk Progress Updates :
We need to show the client how many PDFs out of the bulks are done.

Our Way : 
When a bulk job starts, the API creates a Redis key tracking progress: batch_status:9981 -> { total: 100, completed: 0, failed: 0, status: "processing" }.
Every time a background worker successfully finishes one of the PDFs in that batch, it increments the completed count in Redis.
The frontend client repeatedly pings a status endpoint (e.g., GET /api/bulk/9981/status). This endpoint simply reads the Redis key and returns 45/100 completed.
When the count reaches 100, the status changes to "done" and provides a download link to a ZIP file containing all generated PDFs.

4. Solving the Data Consistency Constraint :
What if, someone edits anyone of the invoice's during bulk-pdf proccessing ? How do we know the data is updated or not ?

Solution :
Do not allow the background workers to fetch live data from the primary PostgreSQL database mid-render.

Instead, mandate that the primary Django application sends the complete, finalized data for all 100 PDFs inside the initial JSON payload. Your microservice must store this exact JSON snapshot in an intermediary database or AWS S3 bucket the moment the request arrives. When the background worker finally reaches the 87th PDF three minutes later, it renders the template using the locked-in snapshot data, completely ignoring any subsequent changes made in the live ERP system.

5. Unreliable Client Connectivity :
Small businesses on poor connections attempt to download a bulk ZIP file containing 50+ PDFs, but the connection drops midway, forcing them to restart the entire download and wasting server bandwidth.

Let's try this :
Resumable Cloud Storage Links
Do not stream the ZIP file directly from your Node.js/Puppeteer server to the client.

Once the background worker finishes the final PDF in a bulk batch, it compresses them into a single ZIP file and uploads it to an object storage service like AWS S3.

The API generates a temporary "Pre-signed URL" pointing to that S3 file and sends the link to the user.

AWS S3 natively supports HTTP Range Requests. If a user's internet cuts out at 60% completion, their browser can use the exact same link to request the remaining 40% of the file without starting over from the beginning.

6. Tamper Evidence : 
The compliance team must be able to mathematically prove that a generated invoice or purchase order was not edited (e.g., someone changing a $100 invoice to $10,000) after it was generated by the microservice.

Optimal way :
Cryptographic Hashing: 
When the PDF is generated, the microservice runs it through a SHA-256 hashing algorithm to create a unique text string (a digital fingerprint). This hash is stored permanently in the PostgreSQL database. If an auditor wants to verify a PDF, they re-hash the file. If the file was altered by even a single pixel, the new hash will not match the database record.   

Trade-off: Cheaper and easier to implement, but requires an auditor to have access to your database to verify the document.

Alternative : 
Digital Signatures: The microservice uses a private cryptographic key to apply a standard X.509 digital signature directly into the metadata of the PDF file itself before saving it.

Trade-off: More complex to set up key management, but any standard PDF viewer (like Adobe Acrobat) will automatically display a "Signed and Valid" green checkmark without needing to query your database.

-----------------------------------------------------------------------------------------------------------------------

## AI CHAT LOG : https://share.gemini.google/ZbXs3h6cAlIp

## Weaknesses & Future Improvements :
Weakest part: Relying on the primary ERP's existing Redis instance for mission-critical queuing. If the monolith crashes Redis, all bulk PDF generation states are permanently lost.

Future Improvements (10x Load): I would abandon Redis for queuing and move to a highly durable, serverless queuing mechanism like AWS SQS. Furthermore, I would migrate the Puppeteer workers to AWS Lambda. Lambda scales to zero when idle and can instantly spin up 1,000 concurrent micro-containers during a massive traffic spike, ensuring fast processing for 10x loads without maintaining expensive idle servers.

## One Thing The Problem Statement Didn't Mention : 
Observability and Alerting via a Dead Letter Queue (DLQ).
The requirements mention failure modes (OOM crashes, dropped connections), but do not mention how engineers are notified when a systemic failure occurs. If a badly formatted HTML template causes 100% of PDF generations to crash, the jobs will infinitely loop or silently fail in the background. Implementing centralized logging and CloudWatch alerts on a Dead Letter Queue ensures engineers are immediately paged when automated invoicing halts, which is critical since broken invoices directly block company revenue.

## Cost Estimation :
API Web Server (Single Requests): 1x AWS EC2 t3.medium (4GB RAM) = ~$30.00
Asynchronous Worker Node (Bulk/Queue): 1x AWS EC2 t3.medium (4GB RAM) = ~$30.00
Object Storage (AWS S3): 30,000 PDFs * ~1MB average size = 30 GB of storage. 
S3 Standard pricing ($0.023 per GB) = ~$0.69   
Data Transfer (Outbound Internet): 30 GB bandwidth out = ~$3.00Database & Cache: $0.00 (Leveraging the existing PostgreSQL RDS and Redis instances per requirements).   
Total Monthly Cost Estimate: ~$63.69 USDThis provides a ~57% buffer against the $150 limit, allowing for auto-scaling an additional worker node during sustained peak events if necessary


---------------------------------------------------------------Thank You----------------------------------------------------------------
