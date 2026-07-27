# BookSwap — Container Diagram

The member uses the React Native BookSwap mobile app, which authenticates with Microsoft Entra External ID and calls the Node.js Express API over HTTPS using a bearer JWT.  
The API handles book, borrow-request, loan, and notification operations, stores correctness-critical business data and transactional outbox events in Azure SQL, and uses Azure Cache for Redis only for selected hot catalogue reads.  
Book photos are uploaded by the API to Azure Blob Storage, while the mobile app downloads the photo files using URLs returned by the API.  
The Background Worker and Scheduler reads unpublished outbox events and digest data from Azure SQL and publishes or consumes notification and digest messages asynchronously through Azure Service Bus.  
After processing queued work, the worker stores persistent in-app notifications in Azure SQL and sends weekly digest emails through Azure Communication Services Email.