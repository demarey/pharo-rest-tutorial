# Pharo Rest Tutorial

## Introduction
In this tutorial, you will learn how to set up a REST backend in a very short time with Pharo.
This tutotial will use some Open Data to populate ther service. Our service will expose the list of Automated External Defibrillator (AED) available to the public in France. Data is provided by OpenStreetMap organisation at https://www.data.gouv.fr/fr/datasets/defibrillateurs-automatiques-externes-issus-dopenstreetmap/. 

## REST
A RESTful web application exposes information about itself in the form of information about its resources. It also enables the client to take actions on those resources, such as create new resources (i.e. create a new user) or change existing resources (i.e. edit a post).
When a RESTful API is called, the server will transfer to the client a representation of the state of the requested resource (any object the API can provide information about). The representation of the state is often in a JSON format but could also be in another format like XML, HTML.
What the server does when the client, call one of its APIs depends on 2 things that you need to provide to the server:
1. **An identifier for the resource** you are interested in. This is the URL for the resource, also known as **the endpoint**. In fact, URL stands for Uniform Resource Locator.
2. **The operation** you want the server to perform on that resource, in the form of an **HTTP method, or verb**. The common HTTP methods are **GET** for reading a resource, **POST** for creating a new resource, **PUT** for updating or replacing an existing resource, and **DELETE**.

For example, fetching a specific Twitter user, using Twitter’s RESTful API, will require a URL that identify that user and the HTTP method GET.

Usage example: get the list of magazines in JSON format
```
GET /api/v1/magazines.json HTTP/1.1
Host: www.example.gov.au
Accept: application/json, text/javascript
```

## Preparation
### Install Pharo Launcher
Pharo Launcher is the fastest way to get a working Pharo environment: image (an object space with Pharo Core libraries) + virtual machine. Pharo Launcher is a tool allowing you to easily download Pharo core images (Pharo stable version, Pharo development version, old Pharo versions, Pharo Mooc) and automatically get the appropriate virtual machine to run these images. 
You can install Pharo Launcher from https://pharo.org/download.
Pharo Launcher documentation is available at https://pharo-project.github.io/pharo-launcher/installation/.

### Set up Data
1. Download the dataset at https://www.data.gouv.fr/fr/datasets/r/e153b245-a704-422b-82c3-95eb6a296a0f. Once unzipped, you will find following files:
    - license.txt
    - metadata.csv
    - data.csv
    
    We will only use the `data.csv`file.
2. Download a fresh `Pharo 10.0 - 64bit` image through Pharo Launcher and launch it
3. We will load a library in our Pharo image to easily import the data.
```smalltalk=
Metacello new
	repository: 'github://svenvc/NeoCSV/repository';
	baseline: 'NeoCSV';
	load.
```
You can find documentation on NeoCSV in **Enterprise Pharo** book: http://files.pharo.org/books-pdfs/entreprise-pharo/2016-10-06-EnterprisePharo.pdf.
4. Clean the data
An AED information can be splitted on many lines (description with line breaks). Let's uniformize the content.
```smalltalk=
csvFile := FileLocator home / 'aed_csv' / 'data.csv'.
"Get lines from the data file without the header"
lines := csvFile contents withInternalLineEndings lines copyWithoutFirst.
" Iterate over lines and merge them when it does not begin with a number"
records := OrderedCollection new: lines size.
lines do: [ :line |
	(line first isDigit or: [ line first = $- ])
		ifTrue: [ records add: line ]
		ifFalse: [ records atLast: 1 put: records last , line ]
	 ].
```

7. 

### Create our domain objects
Since our domain is about AED, let's create an AED object.
We now have one line per AED. We can build our domain objects.
```smalltalk=
records collect: [ :each | | record |
	record := $; split: each.
	AED new
		latitude: (record at: 5);
		longitude: (record at: 6);
		city: (record at: 11);
		freeAccess:  (record at: 13) = 'oui';
		accessInfo: (record at: 17);
		yourself. ]
```

## Basic REST back-end
We will first load *Tealight* library that includes a micro web frameworks as well as small layer to ease its integration into Pharo.
```smalltalk=
Metacello new 
	repository: 'github://astares/Tealight/repository';
	baseline: 'Tealight';
	load
```
You can find some documentation on the project page: https://github.com/astares/Tealight.

### Working with the Tealight server
After you have the framework installed you can easily start a **Tealight** web server by selecting

 _"Tealight"_ -> _"WebServer"_ -> _"Start webserver"_

from the Pharo world menu. Internally this starts a Teapot server with some defaults.

![Tealight menu](https://github.com/astares/Tealight/raw/master/images/Menu.png)
You can also easily stop the server from the Tealight web server menu by using _"Stop webserver"_ or open a webbrowser on the server by using _"Browse"_.

### Accessing the default Teapot
After you started the server you can easily access the running Teapot instance from your code or playground

```Smalltalk
TLWebserver teapot.
```

You can easily experiment with Teapot routes, for instance using

```Smalltalk
TLWebserver teapot 
	GET: '/hi' -> 'HelloWorld'.
```

If you point your browser to [http://localhost:8080/hi](http://localhost:8080/hi) you will receive your first "HelloWorld" greeting.

If you open an inspector on the Teapot instance 

```Smalltalk
TLWebserver teapot inspect.
```

you will see that a dynamic route was added:

![Inspector on the teapot](https://github.com/astares/Tealight/raw/master/images/InspectorTeapot.png)

So you can dynamically add new routes for GET, POST, DELETE or other HTTP methods interactively.

We recommend to read the [Teapot chapter of the Pharo Enterprise Book](https://ci.inria.fr/pharo-contribution/job/EnterprisePharoBook/lastSuccessfulBuild/artifact/book-result/Teapot/Teapot.html) to get a better understanding of the possibilities of the underlying **Teapot** framework.


## Bind the REST server with the domain

We will now try to get the data of an EAD. Let us add a new route that will return a random AED.

```Smalltalk
TLWebserver teapot 
	GET: '/random' -> [ AEDStore default items atRandom ].
```
If you point your browser to [http://localhost:8080/random](http://localhost:8080/random) you will receive "an AED" ...
What happened?
Our webserver receives an HTTP GET request on `/hi` URL. It then executes the associated action, getting a random AED in the store, and then provide it back in the HTTP answer. Objects cannot be transfered through HTTP. They need to be serialized. The default serialization used by our web server is the text representation of the object, i.e. the result of `#printString`method applied to the object.
```Smalltalk
aed := AEDStore default items atRandom.
aed printString  "'an AED'"
```

We now need to define a serialization four our domain object. A widely used format is JSON.
We will add a method `neoJsonMapping:` on the AED class that will specify how to map an EAD to a Json object:
```Smalltalk
neoJsonMapping: mapper
	mapper mapAllInstVarsFor: self.
```
Now, let's take a look at the JSON produced for an AED:
```Smalltalk
NeoJSONWriter toStringPretty: aed. 
```
will return:
```json
{
"id" : 6231,
"latitude" : "48.1739017998959",
"longitude" : "6.44966",
"city" : "Épinal",
"freeAccess" : false,
"accessInfo" : ""
}
```
We can now ask our webserver to encode objects using json as default.
```Smalltalk
TLWebserver teapot
	output: TeaOutput json.
```
You can now point your browser to [http://localhost:8080/random](http://localhost:8080/random) and you will see the difference.
You can now reset the web server routes: 
```Smalltalk
TLWebserver teapot removeAllDynamicRoutes.
```

## Defining a REST based interface
### REST API in annotated methods

While it is nice to experiment with dynamic routes by adding them one by one to the Teapot instance it would be even more convinient
- if we could define the REST API using regular Smalltalk methods,
- if we could map each URL easily.

To support that Tealight adds a special utility class (called TLRESTAPIBuilder) to help you easily build an API. Lets see how we can use it.

First of all we need to create a simple class in the system either from the browser or with an expression:
```Smalltalk
Object << #PharoWorkshopRESTAPI
	slots: { };
	package: 'AA-RestTuto'
```
Now we can define a class side method:
```Smalltalk
random: aRequest
	<REST_API: 'GET' pattern: 'random'>
	
	^ AEDStore default items atRandom
```
As you see we use a pragma in this class marking the class side method as REST API method and defining the kind of HTTP method we support as well as the function path for our REST service.

### Generate our web API
Now we can use the utility class to generate the dynamic routes for us, sending a message to our class ending up in our method:
```Smalltalk
TLRESTAPIBuilder buildAPI 
```
This creates the dynamic routes for us.

To simplify the update of the API and do not loose the server configuration, we can define a class method on `PharoWorkshopRESTAPI` to store the configuration:
```Smalltalk
configuration

	^ TLWebserver defaultConfiguration copyWith: #defaultOutput -> #json
```
Then we will add a `#build` method to simplify the building:
```Smalltalk
build

	TLWebserver defaultServer configuration: self configuration.
	TLRESTAPIBuilder buildAPI.
	TLWebserver start. "ensure the web server is started"
```

Also note that by default, there is an "api" prefix generated into the URL for all REST based methods so you need to point your browser to: http://localhost:8080/api/random.

We can now add a new method to get an AED given its id. Just before, we will define another method that will provide the AED store as most of the api will need it.
```Smalltalk
store
	^ AEDStore default
```
```Smalltalk
aed: aRequest
	<REST_API: 'GET' pattern: 'aeds/<id>'>
	
	^ self store aedWithId: (aRequest at: #id) asInteger
```
Before trying this new route, we need to add the `#aedWithId:` method (instance side) on 
`AEDStore`.
```Smalltalk
aedWithId: anId
	^ self items detect: [ :aed | aed id = anId ]
```
You can notice our pattern now includes an **id** parameter that will be retrieved from the URL.
In a browser, we can now point to: http://localhost:8080/api/aeds/1

We can also add a route to delete an AED:
```Smalltalk
removeAed: aRequest
	<REST_API: 'DELETE' pattern: 'aeds/<id>'>
	
	^ self store removeAedWithId: (aRequest at: #id) asInteger
```
Before trying this new route, we need to add the `#removeAedWithId:` method (instance side) on 
`AEDStore`.
```Smalltalk
removeAedWithId: anId
	^ self items remove: (self aedWithId: anId)
```
To try to remove the AED with id #2, we can use curl command in a terminal.
First, we will fetch the AED with #id 2.
```bash
$ curl -X GET http://localhost:8080/api/aeds/2
{"id":2,"latitude":"42.3143360005073","longitude":"9.2671767","city":"Sermano","freeAccess":false,"accessInfo":""}
```
```bash
$ curl -X DELETE http://localhost:8080/api/aeds/2
```
If you then try to get the resource, you will get a Not Found error:
```bash
$ curl -X GET http://localhost:8080/api/aeds/2
NotFound: [ :aed | aed id = anId ] not found in OrderedCollection
```

### Add a link to the localisation of the AED
We can also provide in our API a way to easily locate an AED on a map.
We can do that quickly by adding a new method on the `AED` object:
```Smalltalk
mapLink
	^ 'https://www.openstreetmap.org/search?query=' , self latitude , ',' , self longitude, '#map=18/' , self latitude , ',' , self longitude, '&layers=N'
```
and by adding a new route on our API:
```Smalltalk
aedLocalisation: aRequest
	<REST_API: 'GET' pattern: 'aeds/<id>/map'>
	
	| aed |
	aed := self store aedWithId: (aRequest at: #id) asInteger.
	^ #map -> aed mapLink
```
Now, you can try to browse an AED localization at http://localhost:8080/api/aeds/1/map. Click on the provided link.

## Navigation (HATEOAS)
So far, we have a web-based service that handles the core operations involving AED data. But that’s not enough to make things "RESTful". In fact, what we have built is better described as RPC (Remote Procedure Call). That is because there is no way to know how to interact with this service. You would  have to write a document to describe its usage.
So what is missing? Hypermedia links to allow resources discovery and navigation. A side effect of NOT including hypermedia in the representation is that clients MUST hard code URIs to navigate the API.
This is what is called HATEOAS: Hypermedia As The Engine of Application State.

### A decorator to add links to our domain objects
We will define a new class that will decorate a class of our domain. It is a simple decorator that will add links to resources to an existing domain object.
Let's create a `HateoasEntity` class:
```Smalltalk
Object << #HateoasEntity
	slots: { #model . #links };
	package: 'AA-RestTuto'
```
It will have a method on class side to easily instantiate it on the decorated object:
```Smalltalk
on: aModel
	^ self new
		initializeWith: aModel;
		yourself
```
On instance side, we define the #initialization method as follows:
```Smalltalk
initializeWith: aModel
	model := aModel.
	links := Dictionary new.
```
We will add a method to easily add a self link:
```Smalltalk
addSelfLink: relativeUrl
	links at: #self put: (self serverUrl / relativeUrl) asString 
```
and a method to get the server base URL:
```Smalltalk
serverUrl 
	^ PharoWorkshopRESTAPI serverUrl
```
`PharoWorkshopRESTAPI class >> serverUrl`is defined as follows:
```Smalltalk
serverUrl
	^ TLWebserver defaultServer teapot server url / 'api'
```
It is the sever url of the default Teapot server we use to serve our REST resources.
We now need to ensure that our `HateoasEntity` objects can be serialized in JSON since the API will now use this decorator to add links to our resources and send it as the answer to the HTPP request. We need to merge the decorator and the decorated object (domain object) into the same structure. We will use a simple dictionary that is easily convetible to JSON.
```Smalltalk
asDictionary

	| dict |
	dict := Dictionary new.
	model class slotNames do: [ :slotName | 
		dict at: slotName put: (model instVarNamed: slotName) ].
	dict at: #links put: links.
	^ dict
```
To avoid the hardcoding of a domain object variables (slots), we use Pharo instrospection capabilities to add decorated object instance variable one by one to the dictionary. Last, we add all the links we want to add to this object.
We also need to define the JSON serialization of the `HateoasEntity` objects. We define the mapping in a `HateoasEntity` class method: 
```Smalltalk
neoJsonMapping: mapper

	mapper
		for: self
		customDo: [ :mapping | 
		mapping encoder: [ :entity | entity asDictionary ] ]
```
Let's now try to generate a JSON entity for an AED with a self link:
```Smalltalk
aed := AEDStore default items atRandom.
NeoJSONWriter toString: 
    ((HateoasEntity on: aed) 
        addSelfLink: 'aed/' , aed id asString;
        yourself) asDictionary
```
Evaluating and printing (CTRL+P) this expression in a playground, you should see something like:
```json
{
    "latitude":"48.1739017998959",
    "city":"Épinal",
    "accessInfo":"",
    "freeAccess":false,
    "longitude":"6.44966",
    "id":6231,
    "links":{
        "self":"http://localhost:8080/api/aed/1"
    }
}
```

### Define a route to get all AEDs
We now have everything to define a route to get a list of ALL AEDs. We will not give all information on AED since data size would be too big. Instead, we will just provide a list of links to discover AEDs.
In PharoWorkshopRESTAPI class, add an `#aeds:` method:
```Smalltalk
aeds: aRequest
	<REST_API: 'GET' pattern: 'aeds'>
	
	^ self store items collect: [ :aed | (self serverUrl / 'aeds' / aed id asString) asString ]
```
For each AED in the store, we collect the link to access the AED resource. This is the list of links that we will send back to the client. Here is a sample output if you try to access http://localhost:8080/api/aeds:
```json
[
	"http://localhost:8080/api/aeds/1",
	"http://localhost:8080/api/aeds/2",
	"http://localhost:8080/api/aeds/3",
	"http://localhost:8080/api/aeds/4",
	"http://localhost:8080/api/aeds/5",
	"http://localhost:8080/api/aeds/6",
	"http://localhost:8080/api/aeds/7",
	"http://localhost:8080/api/aeds/8",
	"http://localhost:8080/api/aeds/9",
	"http://localhost:8080/api/aeds/10"
]
```

You can now click on any resource to perform a new rest request and discover the resource.

### Add a self link to EAD entities
A good practice in REST is to provide a self link to resources that are sent to the client.
We can update our `#aed:` method to include a self link:
```Smalltalk
aed: aRequest
	<REST_API: 'GET' pattern: 'aeds/<id>'>
	
	| aed |
	aed := self store aedWithId: (aRequest at: #id) asInteger.
	^ (HateoasEntity on: aed)
		addSelfLink: 'aeds/' , aed id asString;
		yourself
```

Here is the result of a call to a specific AED:
```json
{
    "latitude":"48.1739017998959",
    "city":"Épinal",
    "accessInfo":"",
    "freeAccess":false,
    "longitude":"6.44966",
    "id":6231,
    "links":{
        "self":"http://localhost:8080/api/aed/1"
    }
}
```
