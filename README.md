# Microservices Demo App

This is a multi component application designed to showcase microservices design patterns & deployment architectures. It consists of a front end single page application (SPA), two lightweight services, supporting database and back end data enrichment functions.

The application is called *'Smilr'* and allows people to provide feedback on events or sessions they have attended via a simple web & mobile interface. The feedback consists of a rating (scored 1-5) and supporting comments.

- The user interface is written in Angular (Angular 5) and is completely de-coupled from the back end, which it communicates with via REST. The UI is fully responsive and will work on on both web and mobile.

- The two microservices are both written in Node.js using the Express framework. These have been containerized so can easily be deployed & run as containers

- The database is a NoSQL document store holding JSON, provided by *Azure Cosmos DB*

The app has been designed to be deployed to Azure, but the flexible nature of the design & chosen technology stack results in a wide range of deployment options and compute scenarios, including:
- Containers: *Azure Container Service (ACS or AKS)* or *Azure Container Instances* 
- Platform services: Regular Windows *Azure App Service (Web Apps)* or Linux *Web App for Containers*
- Serverless: *Azure Functions*
- Virtual Machines: Sub-optimal but theoretically possible

This application supports a range of demonstration, and learning scenarios, such as:
 - A working example of microservices design
 - Use of containers, Docker & Kubernetes
 - No-SQL and document stores over traditional relational databases
 - Development and deployment challenges of single page applications 
 - Platform services for application hosting
 - Using serverless technology to support or provide services
 - Use of an open source application stack such as Angular and Node.js
 - RESTful API design 

---

# :file_folder: Repository Structure
The main levels of the repository directory tree are laid out as follows 
```
/
├── angular            The main frontend Angular app
│   └── src               Angular source code
├── azure              Supporting files for Azure deployment etc
│   ├── appservice        Script to deploy frontend to Azure App Service
│   ├── functions         Azure Functions implementation of the data-api service
│   └── templates         Example ARM templates
├── docs               Documentation
├── dotnet             .NET Core ASP implementation of the services - WIP
├── kubernetes         Docs and files to support deployment to Kubernetes & AKS
├── node               Main microservices, written in Node.js
│   ├── data-api          Data API service source code
│   └── frontend          Frontend service source code
├── orleans            Orleans actor model implementation of the services  - WIP
├── scripts            Supporting helper scripts
│   ├── genOTP            Generate one time passwords for API testing
│   └── initdb            Initialize the Cosmos database and populate with demo data
└── servicefabric      Service Fabric implementation of the services - WIP
```

# Dev Tools & Pre-Reqs
If you are looking to build & work with the Smilr app locally, either as a learning exercise or to run demos, there are a small number of pre-reqs:

- [Node.js](https://nodejs.org/en/download/) installed, if using Windows then [installing Node under WSL is also an option](https://nodejs.org/en/download/package-manager/#debian-and-ubuntu-based-linux-distributions)
- [Docker CE](https://store.docker.com/search?offering=community&type=edition). Required if building the Docker images to run locally, or in Azure or Kubernetes. Note. A remote Docker host can be set-up & used (e.g. via Docker Machine) rather that installing Docker locally. The process of setting this up are outside the scope of this readme
- [VS Code](https://code.visualstudio.com/) This project has been developed entirely using VS Code. However you can use any editor/IDE you wish, but VS Code is highly recommended. Various task & debug configurations for VS Code are provided.

# Architecture & Core App Components
![arch]https://user-images.githubusercontent.com/14982936/32730129-fb8583b2-c87d-11e7-94c4-547bfcbfca6b.png)

The main app components are:
1) [Angular front end UI](#angular)
1) [Frontend service](#front)
1) [Backend data API service](#data-api)
1) [Database](#db)
1) [Optional serverless components](#serverless) 

These will be each described in their own sections below. 

---

<a name="angular"></a>

# Component 1 - Angular Front End UI 
This app was generated with the [Angular CLI](https://github.com/angular/angular-cli) and uses Angular 5.0. To build and run you will need Node.js installed (6.11 and 8.9 have been tested) and also NPM. To install the Angular CLI run `npm install @angular/cli -g`, v1.5.0 or higher will be needed. 

### Development Server
First install packages from NPM by running `npm install`. Run `ng serve` for a dev server. Navigate to `http://localhost:4200/`. The app will automatically reload if you change any of the source files.
When running in non production (or dev) mode, InMemoryDbService is used to provide a mock HTTP API and datastore, this will intercept all HTTP calls made by the app and act as both the API and DB.

### Build
Run `ng build` to build the project. The build artifacts will be stored in the `dist/` directory. You should use the `--prod` flag for a production build.

### API endpoint configuration
***IMPORTANT!*** The API endpoint for the backend data service must be set, there are two ways this is done, depending on if you are running in non production (e.g. from `ng serve`) or in production mode:
- **Non production:** The API endpoint is set in [environment.ts](angular/src/environments/environment.ts), see comments in there for details. ***However in non-prod mode the value of this setting is always ignored*** as the InMemoryDbService intercepts all calls
- **Production mode:** The API endpoint is fetched dynamically at runtime, from the frontend server where it is set as an environmental variable. This is loaded using a call to a special API on the frontend server (see below) by a [ConfigService](angular-src/app\/config.service.ts) which is loaded during app initialization. Note. The static config file [environment.prod.ts](angular/src/environments/environment.prod.ts) controls what variables **ConfigService** fetches

### UI Screenshot
![screen](https://user-images.githubusercontent.com/14982936/32730539-4b85e806-c87f-11e7-89a5-a12543314a34.png)



<a name="front"></a>

# Component 2 - Frontend service
This is held in [node/frontend](node/frontend) and is an extremely simple Node.js Express app. It simply serves up the static content of the Angular app (e.g. index.html, JS files, CSS and images). Once the client browser has loaded the app, no further interaction with this service takes place. This service is stateless

By default the Express server serves the static content using `express.static()` from its root directory (i.e. where `server.js` is), this content should be the output of the Angular build & bundle process `ng build --prod` which outputs to `angular/dist` directory.  
So to package a working frontend service these files must be copied (recursively) across. Note. Do not copy the `dist` directory, only the *contents under it*. You must overwrite the **index.html** file which is just a placeholder.  
The [Dockerfile](node/frontend/Dockerfile) carries out both the Angular build and the copy step so will always create a fully packaged frontend

When running locally you can skip this copy step (see below)

The service listens on port 3000 and requires a single configuration variable to be set. This taken from the OS environmental variables. A `.env` file [can also be used](https://www.npmjs.com/package/dotenv) if present.

|Variable Name|Purpose|
|-------------|-------|
|API_ENDPOINT|The URL endpoint of the data service API, e.g. `https://myapi.azurewebsites.net/api`|

### Front end server - config API
The frontend server presents a special API located at `/.config` this API responds to GET requests and will return values of environmental variables from the server as JSON. This is a workaround to a well known configuration limitation of all client side JS apps (such as Angular, React and others)

The API takes a comma separated list of variable names, and returns them in a single JSON object e.g.
**GET `/.config/HOSTNAME,FOO`** will result in `{"HOSTNAME":"hostblah", "FOO":"Value of foo"}`

This config API is used only once and at startup by the Angular app's **ConfigService** to get the API endpoint from the API_ENDPOINT environment var. The ConfigService is injected in using Angular's APP_INITIALIZER token which suffers from non-existent documentation, however [this blog post](https://www.intertech.com/Blog/angular-4-tutorial-run-code-during-app-initialization/) is a good source of information. This approach allows dynamic configuration of the endpoint address without needing to re-build the Angular app

### Running frontend service/server locally
If you want to run the front-end service locally, you can point the server at a directory containing the static content you want to serve, i.e. the bundled output of `ng build --prod`. To do this pass the directory as a parameter to the `server.js` e.g.
```
node server.js C:\Dev\microservices-demoapp\angular\dist
```
This saves you copying the Angular dist content to same folder as the Node **server.js** file. The path must be fully qualified and not relative.



<a name="data-api"></a>

# Component 3 - Backend Data API Service
This is held in [node/data-api](node/data-api) and is another Node.js Express app. It acts as the REST API endpoint for the Angular client app. This service is stateless

The API routes are held in `api_events.js`, `api_feedback.js`, `api_other.js` and currently are set up as follows:
#### Events:
- `GET /api/events` - Return a list of all events
- `GET /api/events/filter/{active|future|past}` - Return list of events filtered to a given time frame
- `GET /api/event/{id}` - Return just one event, by id
- `POST /api/events` - Create a new event (*secured admin API call*)
- `PUT /api/events` - Update existing event (*secured admin API call*)
- `DELETE /api/events/{id}` - Delete event (*secured admin API call*)

#### Feedback:
- `GET /api/feedback/{eventid}/{topicid}` - Return all feedback for given event and topic
- `POST /api/feedback` - Submit feedback

#### Other helper routes:
- `GET /api/info` - Provide some information about the backend service, including hostname (good for debugging & checking loadbalancing)

### Swagger / OpenAPI
There is a [Swagger definition file for the API](node/data-api/swagger.json) and Swagger UI is also available, just use `/api-docs` as the URL, e.g.  **http://localhost:4000/api-docs/**

### Admin Calls Security
The event PUT, POST and DELETE calls are considered sensitive, and are only called by the admin section of the Smilr app. Optionally these calls can be locked down to prevent people hitting the API directly. For demos it is suggested that the APIs are left open for ease of showing the API and the working app, however for a permanent or live instance it should be restricted.

To switch on security for these three calls, set the `API_SECRET` environmental variable with a key you want to use, note the key can be any length but only contain the following characters: 
`ABCDEFGHIJKLMNOPQRSTUVWXYZ234567`.  
This key is used to generate Time-based One-time Passwords (TOTP), these passwords must be sent to the API in the HTTP request headers, in a header named **X-SECRET**. The password is validated against the key and is only valid for 30 seconds. Invalid passwords will result in a 401 response.

Once enabled the Angular client will need to know this key so it can generate the TOTP to send with any requests to the event PUT, POST and DELETE calls. This is done by setting it in **environment.prod.ts** in the `dataApiKey` field. 

Note. If `API_SECRET` is not set, any value sent in the X-SECRET header is not validated and is just ignored, the header can also be omitted.  
Also Note. The GET methods of the event API are always open and not subject to TOTP validation, likewise the feedback API is left open by design


### Data access
All data is held in Cosmos DB, the data access layer is a plain ES6 class **DataAccess** in [lib/data-access.js](node/data-api/lib/data-access.js). All Cosmos DB specific code and logic is encapsulated here

### Data API server - Config
The server listens on port 4000 and requires two configuration variables to be set. These are taken from the OS environmental variables. A `.env` file [can also be used](https://www.npmjs.com/package/dotenv) if present.

|Variable Name|Purpose|
|-------------|-------|
|COSMOS_ENDPOINT|The URL endpoint of the Cosmos DB account, e.g. `https://foobar.documents.azure.com/`|
|COSMOS_KEY|Master key for the Cosmos DB account|
|API_SECRET|*Optional* secret key used for admin calls to the event API (setting this turns security on, see **Admin Calls Security** above for details)|

### Running Data API service locally
Run `npm install` in the **data-api** folder, ensure the environment variables are set as described above, then run `npm start`



<a name="db"></a>

# Component 4 - Database
All data is held in a single Cosmos DB database called **smilrDb** and also in single collection, this collection is called **alldata**

The collection is partitioned on a key called `doctype`, when events and feedback are stored the partition key is added as an additional property on all entities/docs, e.g. `doctype: 'event'` or `doctype: 'feedback'`. Note. the `doctype` property only exists in Cosmos DB, the Angular model has no need for it so it is ignored.  
Note. This may not be the best collection / partitioning scheme but it serves our purposes, and saves costs

### Deploying Cosmos DB
Deployment of a new Cosmos DB account is simple, using the Azure CLI it is a single command. Note the account name must be unique so you will have to change it
```
az cosmosdb create --resource-group SmilrRG --name smilr-cosmos
```

### Database Initialization 
This has been removed from the API and is now done with the **initdb.js** helper script - [full documentation](scripts/initdb)

### Data Model
There are two main models, one for holding an **Event** and one for submitted **Feedback**, there are also **Topics** which only exist as simple objects nested in Events. **Topics** as entities only exist logically client side, from the perspective of the API and database, there are only **Events** & **Feedback**, so events are always stored & retrieved with a simple serialized JSON array of topics within them

```ts
Event {
  id:     any       // Six character UID string or int
  title:  string    // Title of the event, 50 char max
  type:   string    // Type of event ['event', 'workshop', 'hack', 'lab']
  start:  Date      // Start date, an ISO 8601 string; YYYY-MM-DD
  end:    Date      // End date, an ISO 8601 string; YYYY-MM-DD
  topics: Topic[];  // List of Topics, must be at least one
}
``` 
```ts
Topic {
  id:       number       // int (starting at 1)
  desc:     string       // Short description 
  feedback: Feedback[];  // VIRTUAL PROPERTY - Only hydrated client side when reporting
}
``` 
```ts
Feedback {
  id:       number  // Six character UID string or int
  event:    string  // Event id
  topic:    number  // Topic id
  rating:   number  // Feedback rating 1 to 5
  comment:  string  // Feedback comments
}
``` 


<a name="serverless"></a>

# Component 5 - Optional Serverless Components

There are two serverless components to Smilr, and both are optional

### Data Enrichment - Sentiment Analysis
This optional component enriches data as feedback is sumbitted. It takes any comment text in the feedback and runs it through Azure Text Analytics  Cognitive Services. The resulting sentiment score (normalized 0.0 ~ 1.0) is added to the feedback document in the database.

The is implemented in Azure Functions, using the Cosmos DB change trigger, so that when new items are added to the DB, the function runs and processes them. The Function has an output binding back to Cosmos DB to update the document. The Function is written in C#

Further notes are [included with the code here](azure/functions/sentimentFunction)

### Data API Serverless Version
The data-api service has been re-implemented in a serverless model, this is also using Azure Functions. This component is optional as functionally it is identical to the "normal" non-serverless version of the service. It has been created as a small proof of concept around the idea of using serverless design in a RESTful microservices app, such as Smilr.

The complete API of data-api service has not been replicated, rather a minimum subset has been implemented, in order to get the front end functional and users can submit feedback. The admin part of the API and front end has been omitted.

Azure Functions and Azure Functions Proxies are used to reproduce the same REST API as the regular Node service. There are two Functions, the [eventsAPI](azure/functions/eventsAPI/) and [feedbackAPI](azure/functions/feedbackAPI/), both written in JavaScript and ported from the Node.js code with very little change. 

The exact same [data access library](node/data-api/lib/data-access.js) used by the Node service is used by the Functions version, meaning 100% code reuse without any change.

#### Deploying Serverless Data API
You can deploy the serverless data-api into any Functions App, simply copy the whole of the [azure-functions](azure/functions/) to the App Service, into `wwwroot`

![](https://user-images.githubusercontent.com/14982936/36417631-5e5c4cca-1624-11e8-9e22-65e7ff2e31bd.png)

The NPM packages used by the data-access library to connect to Cosmos DB will need to be installed. After copying all the files to your Function App (including package.json) go into the portal for your Function App, then into 'Platform Features' and 'Console', from there run `npm install`

The [proxies.json](azure/functions/proxies.json) file will need to be modified and the `backendUri` values changed to point to your instance of Functions by altering the "changeme" part. You can also change this using the Functions web Portal in the Proxies section

---

# Deploying Locally

If you are deploying Smilr for the first time, and still getting your head around the various moving pieces, you may want to deploy it initially locally on your desktop machine and set up each of the tiers and make sure they are working correctly, and debug locally if you hit issues.

### Run the Cosmos DB emulator 

There is a Cosmos DB local emulator available for Windows - see  https://docs.microsoft.com/en-us/azure/cosmos-db/local-emulator for details of how to set it up. Use the Data Explorer to ensure the emulator is up and running.

### Run the Backend Data API Service

The CosmosDB emulator listens on https://localhost:8081 and uses a predefined security key "C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==" that you will need to assign to COSMOS_ENDPOINT and COSMOS_KEY before running the data service API.

Initialize the database using the **initdb.js** helper script - [full documentation](scripts/initdb)

**Note.** If want to run the the data-api locally but in a container (e.g. in Docker CE for Windows 10), the Cosmos DB emulator will not allow the connection (you will see connection refused errors) this is because the emulator only binds to loopback (127.0.0.1). The fix is to run the emulator the `AllowNetworkAccess` and `Key` switches, e.g.

```
"C:\Program Files\Azure Cosmos DB Emulator\CosmosDB.Emulator.exe" /AllowNetworkAccess /Key=<YOURKEY>
```
The key needs to be a base64 encoded value of 64 bytes (characters). You can use https://www.base64encode.org/ to create a valid key, or simply re-use the pre-defined key.  
The Cosmos endpoint you specify in the data-api container will no longer be localhost, you will need to use your real local IP, e.g. `COSMOS_ENDPOINT=https://192.168.0.53:8081`


### Run the Front End Service

Remember that you need to build a production version of the service, otherwise the Angular app defaults to the in-memory version and the back end version won't be called.
Don't forget to set API_ENDPOINT to the URL endpoint of the data service API, http://localhost:4000/api.
Check that the system works end to end by browsing to http://localhost:3000 and view the events and create new ones, etc. 

---

# Deploying to Azure
There are many ways to host and deploy Smilr in Azure, the primary ones being:

## Docker & Kubernetes (AKS)
[See the this section for full notes and guides on building the Docker images & running in Kubernetes via AKS](/kubernetes)

## Azure Container Instance
[See ARM templates for deploying ACI + Cosmos](/azure/templates/) and also [building Docker images](docs/docker.md)

## Azure App Service
See [PowerShell script](etc/deploy-appsvc/) to build the front end service and deploy to Azure App Service. The backend service can also be deployed as a regular Node app to App Service.

## Service Fabric
Coming soon

## Orleans 2.0 in Containers
Coming soon

---

# Appendix - Running A Secured Instance
The application is designed to be deployed in a demo scenario, so access to the admin pages where you can create/edit events and view feedback is open to all, without login. 

However should you want to run the app permanently in non-demo instance for real use, there is an option to secure it. In `environment.prod.ts` change the `secured` setting to `true`. The Angular app will now hide the admin and report sections behind a login prompt. Authentication is expected to be handled by Azure App Service authentication and AAD.

**Important Note!** For this `secured` mode to work the Angular app must be deployed with the supplied frontend server ([node/frontend](/node/frontend)) and that ***must be deployed to a Windows App Service***. On that App Service, Authentication must be enabled but ***anonymous access should be allowed***, and AAD should be set up as an auth provider. No other configuration will work.

E.g.  
![app-svc-auth](https://user-images.githubusercontent.com/14982936/32653292-4a065c4a-c5ff-11e7-8d48-a3d2df805b4a.png)
