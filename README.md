# Hospital-NoSQL-data-store
University project for the Advanced Data Management course, University of Genoa. The aim of Project is to design and develop a NoSQL data store to support a large scale application.
<br><br>

# Domain and Application 
**Hospital Billing & Information System** focusing on managing patient records, tracking 
encounters, procedures, facilitate billing and insurance claims with payer data.  
# Nature of Application, System Requirements 
Application is mostly used by the hospital and admission staff to control patient’s billings and 
issued procedures. It uses both, **transactional and analytical operations**, so it is **read/write 
intensive** and it needs to support real-time updates to patient records, procedures they had 
and their costs. Frequent reads for medical history as well as visit details (history of procedures), 
billing records including total costs of encounters, coverage by insurance (or its lack). 
<br><br>Additionally, **batch processing** will be needed for tasks like generating hospital performance 
reports (yearly, monthly hospital income, average income per patient, average encounter time, 
types of encounters performed). <br><br>
While **high consistency** is crucial and for us, it is placed in the first place, **availability** of the 
system should also be mediated. Admission staff, receptionists need real-time data on the 
patient's current treatment – need to know current costs of treatments, to have accurate 
calculation of total costs (soon after doctor adds some procedures, or insurance company covers 
some value). Doctors and nurses need to see up to date patient procedure history. While it is 
not necessary as the consistency, system should provide some availability to update and write 
new records to the system.

# Workload and frequent operations 
- What procedures has a patient with a given Codice Fiscale undergone?
- What is the total amount of money that a given payer in given month must cover for encounters?
- How many times in a given month was it necessary to perform 'urgent care'? 

# Chosen system: MongoDB 
As we place high consistency at the first place in our application as the back-end system we 
would use MongoDB. Considering the features of application and identified workload it would 
be the most suitable system for keeping critical data (patient records, procedures, and billing) 
that considers the importance of data accuracy.<br><br>
MongoDB is optimized for frequent reads and writes, essential for updates and querying 
historical patient data. <br><br>
However, the main advantage of MongoDB over other NoSQL systems is that **all nested 
document components can be accessed during those read and write operations**. The 
structure of an aggregate is visible at the logical and application level (useful for operations like 
adding a new procedure to a patient). <br><br>
With this system, application will provide eventual consistency at replicas at the same time 
addressing availability with replica sets, making it fault-tolerant and operational during node 
failures (taking care of high-importance and sensitive data).

# Schema for selected system
![schema](https://github.com/user-attachments/assets/5f2d61e0-fdd3-4d9d-9dbe-43951cad0fc9)
<br>**Q1:** <br>
patients: { _id, CodiceFiscale, hadProcedures:[{ProcedureCode, Description}] } 
<br>Selection Attributes Q1: {CodiceFiscale} 
<br>Shard Key (+unique index): {CodiceFiscale}, hashed 
<br><br>**Q2, Q3:**<br> 
encounters: { _id, EncounterID, Stop, EncounterClass, PayerCoverage, PayerName } 
<br>Selection Attributes Q2: { Stop, PayerName}  
Selection Attributes Q3: { Stop, EncounterClass }  
Shard Key (+ non unique index): { Stop }, hashed 
<br>Unique index: { Stop, EncounterID } 
<br><br>Operations from workload in MongoDB: 
<br> **Q1:** <br> 
```
db.patients.find({"CodiceFiscale":"CF002J6UIMM"},
{"hadProcedures.ProcedureCode":1,"hadProcedures.Description" : 1, "_id":0}).pretty()
```

**Q2:**
<br>
```
db.encounters.aggregate([ {$addFields:{ "Month":{$dateToString:{format:"%Y-%m",date:{$toDate:"$Stop"}}}}},
{$match:{"PayerName":"Medicare", "Month":"2011-01"}},{$group:{_id:"$PayerName",toPay:{$sum:"$PayerCoverage"}}}]);
```
**Q3:**
<br> 
```
db.encounters.aggregate([{$addFields:{"Month":{$dateToString:{format:"%Y-%m",date:{$toDate:"$Stop"}}}}},
{$match: {"Month":"2011-02", "EncounterClass":"urgentcare"}}, {$count: "urgentCareCounts"}]);
```

# Operations of the workload in Neo4j
![neo4j](https://github.com/user-attachments/assets/3277693b-9e28-491b-8324-229c6f37a661)
**Q1:**
<br> 
```
MATCH (p:Patient)-[:hadProcedures]->(proc:Procedure)  
WHERE p.CodiceFiscale = "CF002J6UIMM"  
RETURN proc.ProcedureCode, proc.Description;
```
**Q2:**
<br>
```
MATCH (payer:Payer)-[:paidFor]->(enc:Encounter)   
WHERE payer.PayerName= "Medicare" AND datetime(enc.Stop).month = 1 AND 
datetime(enc.Stop).year = 2011   
RETURN SUM(enc.PayerCoverage) AS TotalCoverage;
```
**Q3:**
<br> 
```
MATCH (enc:Encounter) 
WHERE datetime(enc.Stop).month = 1 AND datetime(enc.Stop).year = 2011 AND 
enc.EncounterClass = "urgentcare"  
RETURN COUNT(enc) AS UrgentCareCount;
```

# System configuration needed in MongoDB for storing and processing

- **Partitioning (sharding):** Hashed-based (hash function on shard keys) 
- **Replica Set:**
  - **Master-slave approach** (one master node - receiving requests, two slave nodes)
  - **Fault-tolerace:** use of primary replicas (accepting read, write operations), secondary replicas (only read operations) 
- **Consistency, Availability:** **CP** Theorem, but trade-off consistency for some availability (loosening some of consistency requirements and making writes on a **QUORUM** of replica nodes, or even making system asynchronous but we don’t think it would be that necessary in this case) 
- **Indexing:** Ensure **indexes** (CodiceFiscale) and **compound indexes** (Stop, EncounteID) on shard keys. 
- **System transactions:** **ACID** - single write operation will modify multiple documents, and modification at each document is atomic (on a level of a single document).
- **Each node Resources**:
  - **Storage:** about 1T
  - **CPU:** Multi-core processor (6-8 cores), handling concurrent operations.
  - **RAM:** 16GB (for real time updates)

Our database is much smaller than (we assume) it would be for a real data-store.
# Implementing schema in system MongoDB. 
**Q1:**
<br>  
```
db.patients.createIndex({“CodiceFiscale”: 1}, {unique: true}) 
db.adminCommand({shardCollection: “db.patients”, key: {“CodiceFiscale”: 1}, field: “hashed”})
```
**Q2,Q3:**
<br>
```db.encounters.createIndex({“Stop”: 1, “EncounterID”: 1}, {unique: true}) 
db.encounters.createIndex({“Stop”: 1}, {unique: false}) 
db.adminCommand({shardCollection: “db.encounters”, key: {“Stop”: 1}, field: “hashed”})
```

# Creating instance of schema in the MongoDB 
[Dataset source: MavenAnalytics](https://mavenanalytics.io/data-playground)<br><br>
Importing files to local machine: 
```
scp "encounters.json" user@XXX.XXX.XX.XX:
```
```
scp "patients.json" user@XXX.XXX.XX.XX:
```
In mongo db we created indexes: 
```db.patients.createIndex({“CodiceFiscale”: 1}, {unique: true}) 
db.adminCommand({shardCollection: “db.patients”, key: {“CodiceFiscale”: 1}, field: “hashed”}) 
db.encounters.createIndex({“Stop”: 1, “EncounterID”: 1}, {unique: true}) 
db.encounters.createIndex({“Stop”: 1}, {unique: false}) 
db.adminCommand({shardCollection: “db.encounters”, key: {“Stop”: 1}, field: “hashed”})
```
Since we want to import data from the existing collections, and we are not allowed to 
partition/shard a collection (admin privileges are needed to enable partitioning, we use university provided server) 
<br><br>Importing data from the json files to populate the collections: 
```
mongoimport  --username=user --password=*** --collection=encounters --db=user_db --file=encounters.json --jsonArray
```
```
mongoimport  --username=user --password=*** --collection=patients --db=user_db --file=patients.json --jsonArray
```
To verify data:
```
db.patients.find().pretty();
```
```
db.encounters.find().pretty();
```

❗**Please find the documentation of the developed data store in 'PROJECT_ADM_MartynaMajewska_MonikaMazella.pdf' file**
