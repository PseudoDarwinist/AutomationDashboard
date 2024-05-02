# AutomationDashboard

https://cloud.msg.flysas.com/general_test

Automation Dashboard
The first thing to address is, what is it? Although similar to the Overview page in Marketing Cloud Automation Studio, this app will let you see more details in a much cleaner way as well as provide additional information such as status, next run date, type, the duration of the last run, and the modified date. It also allows for a better overview with a quicker search bar as well as filtering to only allow a view into automations that have been run in the past 6 months.

This also is a stepping stone, a basic model of the possibilities this dashboard offers. You can even offer up the possibility of using this as an overview for all business units, the ability to take snapshots of your automations in a moment in time, and to provide further details and error information. This can include things such as average run time, total errors and successful runs, and more. The following screenshot gives a visual of what the basic Automation Dashboard looks like.


As you may have noticed, this is a lot different than what you see in Automation Studio. This, although a bit less pleasing visually, is a much easier-to-digest data dashboard to help you easily sort or search through to find the overview info you need. The following gives an explanation of what each column represents:

name: The name given to the automation.
status: The overall status of the automation. This is not in relation to the last run status. This will not show an error if the last run was unsuccessful.
ModifiedDate: The last time the automation was edited.
LastRunDate: The last time the automation was run.
NextRun: The next time the automation will run. This will be empty if there is no new run scheduled.
Stepcount: The number of steps inside the automation. This is the number of steps and not the activities in each step. So, if you had 5 activities in 1 step, it would only show a value of 1 here.
Type: This displays the type of automation. The two available values are Scheduled and Triggered.
Lastrunstatus: This gives the result of the last run. This includes Error, Complete, and Stopped.
Duration: The length of time it took the last time the automation was run.
You may have noticed the names in Figure 8.7 are blue. This is because they are linked. When you click on a name, it will take you, in a new window, to that automation in Automation Studio. If you want to make any edits, you can just click from this dashboard instead of leaving here and going into Automation Studio and finding that automation again.

As you can see, there can be a lot of useful information in the palm of your hand all within the Marketing Cloud. We like to view this as more of a tracking- and reporting-oriented dashboard for those who want to track automations and view their status without needing to interact with them directly.

As you may have noticed, there is a button at the top left that is labeled Export Data. That's right – through some fancy JavaScript, you can click there, and it will download a CSV file of the information used to fill in this dashboard. The CSV file will also contain the AutomationID as well as the ClientId (Business Unit or MID) of the account.

Now that we have a good overview of what the finished product is going to be, let's start exploring the code.

The full code
If you follow the preparations from the previous section and add the appropriate URL and application information into this code, you will be able to pretty much copy and paste this into CloudPages and be up and running.

But you are reading this book because you want to learn, not just copy someone else's code. So, we are going to use the next section to go into detail about the code and how it works. The code is split into a few sections, and we shall go through it accordingly. The split is of HTML, SSJS, and client-side JavaScript.

As it is the first part of the code as well as the first part processed on the page, we will start by exploring the SSJS.

The SSJS
We are going to break this part into different sections. There are four major parts of the SSJS that we will go over. The first is the validation and security to ensure that the referrer is actually coming from Marketing Cloud and that the page is secure. Next is the authorization and authentication to create an OAuth token for the REST API. After that is the calling of the REST API to get the JSON with all automations and then parse and prepare it for use in the client-side JavaScript. Lastly, there's the list of the functions required to make the other three sections work.

Security headers and validation
The first section is dedicated to ensuring that the page you make is unable to be used outside the environment you want it to work in. For this reason, we added a few security headers as well as a conditional to check the referrer URL to validate the source of the request. See the following code snippet from the full code hosted on GitHub:

  //Security headers to secure the page

  HTTPHeader.SetValue("Access-Control-Allow-Methods","POST,

    GET");

  HTTPHeader.SetValue("Access-Control-Allow-Origin","*");

  Platform.Response.SetResponseHeader("Strict-Transport-

    Security","max-age=200");

  Platform.Response.SetResponseHeader("X-XSS-

    Protection","1; mode=block");

  Platform.Response.SetResponseHeader("X-Content-Type-

    Options","nosniff");

  Platform.Response.SetResponseHeader("Referrer-

    Policy","strict-origin-when-cross-origin");

  //Gather the referrer url

  var referrer = Platform.Request.ReferrerURL;

  //Verifies that this site is being called from within

  //SFMC

  //Pay attention to this as if they change the domain,

  //this will break and toss an error

  if (referrer.indexOf('exacttarget.com') < 0) {

    throw error;

  }

To be honest, most of those headers are not necessarily needed in this case, but it is a best practice to include them every time it is possible as it is usually better to be safe rather than sorry. That being said, if you run into errors that you are not understanding, feel free to explore and investigate whether the issue is stemming from these headers (usually you can see if this is the case inside the console of the browser's dev tools).

The main part that is needed is the validator on the referrer URL. This ensures that if the page gets traffic from anywhere outside of the SFMC UI it throws an error. As you can see, it pulls in the referrer URL via SSJS pulling from the Request object. This will return https://sX.exacttarget.com with the X being the stack – except for stack 1, which will remove the s subdomain.

After we collect that URL, we then run an indexOf() function on it to verify that it has exacttarget.com inside of it – used to remove the dynamic stack aspect so it will work across all stacks. Now, at the time of writing this, that was the URL, but we know Marketing Cloud has been working very hard to remove all domains and occurrences of Exact Target still existing in the tool – so in the near future, this may need to be updated to account for this change.

After the comparison we have it throw an error if exacttarget.com does not exist in the referrer URL. This will force the page to return a 500 error and not allow access at all. Next, we will investigate the API authentication portion of the scripting.

API authentication and OAuth token
In order to utilize the REST API to gather the automations, you need to be able to authenticate your call. Now, you could create a new server-to-server API component and utilize that inside of here instead, but that would limit it to just the MID it is created in. But by utilizing the web app package we created, we can use it across each business unit that it is authorized inside of.

In this example, we have the app information and REST API info, including the ID and Secret hardcoded inside of the page, but for enhanced security you can store it elsewhere and encrypt it accordingly. Prior to that information coming into play though, we first need to validate whether it is authorized by reviewing the state parameter:

//App information, including REST API info

  var appURL = '{{urlOfCloudPage}}';

  var clientID = '{{clientID}}';

  var clientSecret = '{{clientSecret}}';

  var tenantID = '{{tenantSubDomain}}';

  //Gather state parameter from URL

  var state =

    Platform.Request.GetQueryStringParameter('state');

  

  //Validate if authorized

  if (state) {

    var code =

      Platform.Request.GetQueryStringParameter('code');

    var payloadObj = {

        grant_type: 'authorization_code',

        code: code,

        client_id: clientID,

        client_secret: clientSecret,

        redirect_uri: appURL

    };

    //Gets Token if already Authorized

    var res = HTTP.Post('https://' + tenantID +

      '.auth.marketingcloudapis.com/v2/token',

      'application/json', Stringify(payloadObj));

    var resJSON =

      Platform.Function.ParseJSON(res.Response[0]);

    var accessToken = resJSON.access_token;

    var authToken = 'Bearer ' + accessToken;

  } else {

    state = GUID();

    Platform.Response.Redirect('https://' + tenantID +

      '.auth.marketingcloudapis.com/v2/authorize

      ?response_type=code&client_id=' + clientID +

      '&redirect_uri=' + appURL + '&state=' + state);

    //Authorizes request if not already authorized

  }

Basically, the flow of this section is as follows:

Set variables for each of the required data points for authorization.
Validate that there is an existing state in the URL.
If a state exists, then we gather the authorization code and send an API call to get the access token.
If a state does not exist, then create a new random value for the state and redirect to the authorize endpoint.
This redirect will validate if the user is logged into SFMC and authorize and provide a code for use on the token call when it redirects back to the original URL.
The first two parts are fairly self-explanatory and you likely do not need us to go into too much explanation. It is just the setting of variables and then a conditional to check whether a value exists in the state variable we just set. The actions inside of this conditional are where we will need to go into details to ensure understanding. First, we will start with a scenario where state exists, and the condition returns true.

Validates as true
So, the first thing that happens after validating as true is that we set the variable of code based on the query parameter, also named code, to get this value. This value is very important as it is required for us to make the token call for the OAuth token we need for authorization.

After we have this value set, we then create a new JavaScript Object variable, named payloadObj. This object will contain all the correct key and value pairs required for us to use to send to the token endpoint and return an access token.

You will notice that this object is different from those we explored in the previous chapter with the server-to-server API component. This one, grant_type, is now an authorization code, and it requires two new keys: code and redirect_uri. Code is filled from the query parameter that we assigned a variable earlier and redirect_uri is the appURL variable we defined at the very top of this section. The two remaining keys are client_id and client_secret, which are used exactly as they are in the server-to-server API call.

Now that we have created the object, we now need to make a POST call, utilizing the HTTP.Post Core function, to the right endpoint and pass payloadObj. Note that the object needs to be stringified, which basically means it needs to be changed from a data type of Object to a data type of String.

After the call is made, we then parse the response and then grab the access_token value from the returned JSON, and then we create a new variable, authToken, that then combines the bearer and the space in front of it to create our finalized auth token. This then gets us ready to make any future REST API calls using this token. Next, we will explore what happens if this condition does not validate as true.

Does not validate as true
Although this section is only two lines of code, there is a lot that goes into it. The first line of code is fairly simple as it just assigns a random Global Unique Identification (GUID) to the state variable for use in the condition in the next run. This uses the literal GUID() function to create it. The next part is the part that requires a bit of extra explanation.

After setting state, we then do a Redirect server side that sends the browser to a new URL instead of the current one. Now, people might wonder why we would want to do this. Well, that is because, in order to authenticate it, we need to go through the authorize endpoint to validate the user is logged into Marketing Cloud and correct code is assigned. So, in order to do this, we need to redirect to that URL and append a few query parameters to the URL to help the endpoint correctly finish its process and redirect back to the page.

As stated, this redirect will actually lead to redirecting back to this page. Once you are logged in and a code has been assigned, the page will be redirected back to the original CloudPage URL you are using, but now with a state query parameter as well as a code value. This will then trigger the previous validates as true process.

We now have the authentication process down and understood, so let's move on to the retrieval and parsing of the response section of the SSJS code.

Retrieving and parsing the result
We are going to break this section into three different sections to make it easier to digest. We will first have the retrieval process, where we set up our variables in preparation for the retrieval, followed by the actual function call – the function itself will be gone over in the later sections that go into details on the functions. Next, we will go over the looping through this JSON to iterate appropriately through each dataset. Then, finally, we will go over the actual setting of the values inside this loop to fill a new object that is then pushed into a new array. Let's dig into the retrieval functions and setup!

Retrieval
This section is actually fairly simple in terms of both the small amount of code and the fact that it is pretty easy to understand. This basically gets a bowl and then goes over and gets the cereal and milk and pours it in. It is a bit more involved than that, but essentially that is what this section is doing. See the following code snippet for a reference on the block we are discussing:

  //Sets as global var for final JSON variable

  var data = [];

  //set Expire value for auto lookback

  var expire = new Date()

  expire.setMonth(expire.getMonth() - 5)

  //5 months subtraction because month is 0 index, meaning

  //0 is January in getMonth, but setMonth will see

  //Jan as 1.

  var autos = getAutomations(tenantID,authToken);

  var items = autos.entry;

The beginning is setting up the data array global variable to be filled through the iteration of each loop in the following sections – or the bowl that we are going to pour the cereal and milk into. We then also set up the expire variable, which is set to be a maximum of 6 months in the past. This is to help us select only those that are relevant in the returned array of automations.

After this, we make the call to the getAutomations()function. This function will then run through a script that makes an API to call and retrieve JSON that contains all the automations inside of that specific business unit. This is the milk and cereal part. After that, we then set up the next couple of variables to begin the looping and iteration through the returned object.

Looping and iteration
Now that we have the data in JSON format, we have each of the automations stored in their own object inside of an array. But how the heck do we get this data out? This is where looping and iteration come into play.

There are multiple types of looping available in SSJS, much more than just the FOR loop that is available inside of AMPscript. The following are some examples of looping available inside of SSJS:

For loop: Repeats until a specified comparison evaluates to false. Here's an example: for (var i=0; i<arr.length;i++) – this means the loop will repeat until i iterates to the same number as the length of the array (arr).
For...in loop: Iterates a specified variable over the enumerable properties of an object. This means that for each distinct property, this loop will run. For example, for(var in Object) – this will run for each instance of var in the Object.
Do...while loop: Repeats until a specified condition evaluates to false. The statement inside this loop is always executed once, but the repetition is determined by the condition set at the bottom. Here's an example: do { //myStuff } while(stuff > 10) – this will do the statements in the do part while the stuff variable is greater than 10.
While loop: Executes the statements as long as the specified condition evaluates as true. A while loop is identical to a do...while loop but without the guaranteed first execution. For example, while(n < 3) { n++; } – this will loop through until n is 3.
For our needs, we are going to be utilizing the while loop to loop and iterate through our array of objects to form our output. This part may be confusing though because our output is going to be another array of objects, so why do we need to manipulate this? Simple: because the layout of the original is not something that is easily digested by our dataTables plugin for jQuery.

We will go over this more in the section ahead that talks about the client-side scripts and code (Client-side JavaScript). Following is a snippet, removing the setting of variables and iteration parts, that shows the loop and the appropriate loop settings and usage:

  var c = 0;

  while(c < items.length) {

    var a = items[c]

     c++;

  }

  //Stringify for use in client-side JS

  var json = Stringify(data);

WHILE LOOP VARIABLE

As you can see from this, c is set outside the loop as 0 to make the comparison valid, comparing it to the total size of the array of objects. c is then used to pull a specific object inside that array and then at the end of the loop, we use c++ to increase the value of c by 1.

Setting appropriate variables
The next part of this that we are going to dig into is setting up the appropriate variables to use when building and parsing out the automation information. Most of this is centered around getting the total runtime from the start and completed times of the automation. See the following snippet of the code for reference:

    //Auto Object for pushing into data array - resets to

    //null on each loop

    autoObj = {};

    var start = new Date(a.startTime)

    var completed = new Date(a.completedTime)

    var delta = Math.abs(completed.getTime() –

      start.getTime()) / 1000

    // calculate (and subtract) whole days

    var days = Math.floor(delta / 86400);

    delta -= days * 86400;

    // calculate (and subtract) whole hours

    var hours = Math.floor(delta / 3600) % 24;

    delta -= hours * 3600;

    // calculate (and subtract) whole minutes

    var minutes = Math.floor(delta / 60) % 60;

    delta -= minutes * 60;

    // what's left is seconds

    var seconds = Math.floor(delta % 60);

    var diff = (hours > 0 ? hours + 'hrs ': '') + (minutes

      > 0 ? minutes + ' mins ' : '') + seconds + ' secs';

AUTOOBJ DECLARATION

Before getting into the calculation for the total runtime, we want to point out, at the top, the variable declaration of autoObj. This is important not only because it sets the data type for this variable, but also because it makes it so that, every loop, it completely clears out the autoObj variable so that there is no accidental duplication across each iteration from data being carried across.

For the calculation of duration, we first start by getting the start date and time and the completed date and time from the JavaScript object, a, which was set in the while loop snippet.

From those two, we have to then get the delta, which is the time in milliseconds between when the automation started and when it finished.

From there, we then use math functions to multiply and divide to get the days, hours, minutes, and seconds difference.

Now to ensure each is accurate, we are going to be removing the previous value, days for hours, hours for minutes, and so on from the delta after the calculation. This ensures we get an accurate number at the end.

Now that we have the correct duration, we just need to put it together, which is what we use the diff variable for. Essentially, in this, we do an inline conditional that checks to see if each section – hours, minutes, or seconds – has a value and if so, then we add that value plus a label. If it does not have a value, then we do nothing.

Iteration and manipulation
Now that we are prepped and have the duration ready, we just need to push all the other aspects of the automation into the proper format in this object and push it into the new array. The following snippet shows how we will do that:

    //Validates if the automation was run in past 6 months

    //Or the automation was modified in past 6 months and

    //was not run but has schedule type defined.

    if ((new Date(a.lastRunTime) > expire) || (new

      Date(a.modifiedDate) > expire && a.lastRunDate ===

      undefined && a.automationType != 'unspecified')) {

     //set each date into a date data type

     var modDate = new Date(a.modifiedDate)

     var lastRunDate = new Date(a.lastRunTime)

     var nextRunDate = new Date(a.scheduledTime)

     //add each key/value pair into autoObj

     autoObj.AutoID = a.id;

     autoObj.ClientID = a.clientId;

     autoObj.Name = a.name;

     autoObj.Status = a.status;

     autoObj.modifiedDate = formatDate(modDate);

     autoObj.LastRunDate = a.lastRunStatus === undefined?

       '' : formatDate(lastRunDate);

     autoObj.StepCount = a.processes ? a.processes.length :

       0;

     autoObj.Type = a.automationType;

     autoObj.LastRunStatus = a.lastRunStatus;

     autoObj.lastRunDuration = diff != '0 secs' ? diff :

       '';

     autoObj.NextRun = formatDate(nextRunDate);

     //push autoObj into the data array

     data.push(autoObj);

    }

SIX-MONTH LIMIT FILTER

This section starts with a conditional statement that will validate if the automation has been run in the past 6 months or if it was recently built, in the last 6 months, but not run. This is to help limit the returned automations to only those that are relevant. It also accounts for a limitation in the API that for automation interactions, a separate call to find details on the specific running of that automation is only available via that endpoint for 6 months. Anything beyond that returns an error. Our rationale for this is to display only those automations that are relevant and reduce the noise displayed.

After this, we then utilize the dot notation on the source object set in the prior scripts, the JavaScript object a, to then assign the new key/value property in the new object, autoObj. When using the statement autoObj.AutoId = a.id, we create the parameter key of AutoId inside of the autoObj and then set the value of it to a.id. Or, for a visual, see the following example:

AutoObj = {

  AutoId: a.id

}

This is then utilized for all the following properties to correctly assign the values to the new properties in autoObj. Then, once we have all these property key/values matched, we use the push() function to take this object and add it into the array, data, which we set previously. This will be the container holding all of the objects of our automations. Next will be a deep dive into the getAutomations() function and how it works.

The getAutomations() function
This function is the key to the whole page. Without this function, we would have no data for the rest of the code to work with. The function utilizes the undocumented REST API endpoint to bulk retrieve automations. As always, you should be wary of using undocumented endpoints for production or live applications, but as this is for internal reference, it should be no issue.

UNDOCUMENTED REST API DISCLAIMER

Salesforce Marketing Cloud's undocumented REST endpoints or SOAP objects are to be used at your own risk. These are unofficial capabilities or access to the platform that at any moment could be altered, removed, or otherwise manipulated, which could break your applications or scripts utilizing them. The other consideration is that as these are not public-facing, the throughput, bandwidth, uptime, and speed may fluctuate or be flimsier than normal, making it an extremely risky solution if used at volume or high frequency. To this extent, it is never recommended to use these endpoints or objects in anything that could have a potential negative impact on your production or live campaigns or processes.

This endpoint exists in the legacy endpoint family. It is actually super-fast considering the significant size of the payload it returns. This endpoint does have a timeout and limitation so if your business unit has an enormous number of automations, it could break and cause this application to error, but we have found this endpoint and scripting to be able to handle all but the most absurd amounts of automations. This is one of the few endpoints that is officially labeled as bulk, and although we have no objective insight into if this is actually a bulk endpoint but, the performance seems to speak for itself.

As a note, the endpoint will return a page size with it, but this page size does not matter as it will always return the full number of automations no matter what value is put in there. The returned array is also formatted differently than most other documented APIs, so it will take a little bit of exploring and experimentation to get the nuances right.

The following is an example of the REST API call to the bulk automation retrieval endpoint:

GET /legacy/v1/beta/bulk/automations/automation/definition/

Host: {{tenant_subDomain}}.rest.marketingcloudapis.com

Authorization: Bearer {{authToken}}

Content-Type: application/json

As you can see in the preceding snippet, it's a very simple endpoint and it only requires the tenant subdomain and authentication token, like most other REST endpoints. One thing that many find alarming is the part inside of it that says beta. We know, this scared us too at first, but this endpoint is utilized inside of the UI of Marketing Cloud, so we think it is fairly safe to assume it is not going to be going anywhere in the immediate future.

To use this endpoint, we are going to be pulling in Script.Util.HttpRequest(), which we went over in Chapter 7, The Power of In-Step APIs, to handle our API calls. You can see the full function in the following snippet for reference:

  //gets a JSON of all automations inside of Business Unit

  function getAutomations(tenantID,authToken) {

    var url = 'https://' + tenantID +

      '.rest.marketingcloudapis.com/legacy/v1/beta/

      bulk/automations/automation/definition/';

    var req = new Script.Util.HttpRequest(url);

    req.emptyContentHandling = 0;

    req.retries = 2;

    req.continueOnError = true;

    req.contentType = "application/json"

    req.setHeader("Authorization", authToken);

    req.method = "GET";

    var resp = req.send();

    var resultStr = String(resp.content);

    var resultJSON =

      Platform.Function.ParseJSON(String(resp.content));

    return resultJSON;

  }

This is a pretty boilerplate way of using the Script.Util.HttpRequest() function by setting the URL, the method, and other properties, sending it out, and then turning the response into a string first, then parsing it as JSON. This function then just returns resultJSON to the variable that is being used to call the function – pretty straightforward.

Now, some of you may have noticed that there is a second function as well. This one is also very important but not as flashy or glamorous as the getAutomations() function. It is the formatDate() function, which we use to prepare the date for display on the dashboard.

This function basically just takes the date type that is normally used and turns it into a string version that is more readable to the average viewer and appears cleaner. You can check out the script for this function in the following snippet:

  //prepares the date for display in dashboard

  function formatDate(myDate) {

    if(typeof(myDATE) !== null) {

    var month = (myDate.getMonth() + 1),

    day = myDate.getDate(),

    year = myDate.getFullYear(),

    hour = myDate.getHours(),

    mins = myDate.getMinutes(),

    meridiem = '';

    if (hour > 12) {

      hour = Number(hour) - 12;

      meridiem = 'PM'

    } else {

      meridiem = 'AM';

    }

    return ('0' + month).slice(-2)  + '/' + ('0' +

      day).slice(-2)  + '/' + year  + ' ' + ('0'

      hour).slice(-2)   + ':' + ('0' + mins).slice(-2) + '

      ' + meridiem;

    } else { return '';}

  }

Most of this is pretty straightforward, where we validate that it is a date, then parse it out to get the month, date, year, hour, minutes, and others as their own variable. You might note that month has it adding one to the value. This is because getMonth is based on a zero index, meaning that January would be returned as 0 and not 1. So, to make it easy to read, we just add one to it.

After that, we validate the hours to see if it's AM or PM and then adjust accordingly. This is because the datetime object pulled from the source object is in the 24-hour format, and not 12. We then move on to the return value, which is a bit of fancy footwork to create a date such as 11/12/2021 4:51PM. Now that we have the SSJS part under our belts and we have the data together and ready to be used, let's explore the HTML and CSS framework and styling we set up.

The HTML and CSS
Admittedly, there is not much HTML or CSS needed here as most of it is pulled in through plugins or libraries. This can be a bit of a bloat and can slow things down, so for sure there are better ways to optimize this, but we have found the following structure and framework to be the easiest to work with while having the least frontend experience needed but still displaying nicely. Following is the snippet of the HTML and CSS needed for this. You will notice at the bottom there is an exorbitant amount of external script libraries, which is what is needed for the framework and dataTables plugin:

<!DOCTYPE html>

<html lang="en">

  <head>

<!-- Required meta tags and external CSS Style Sheets -->

  </head>

  <body>

    <div style="width:90%; padding:20px 0 20px 0; margin:0

      auto;">

      <button id="exportBtn" onclick="download()">

        Export Data</button>

      <table id="autoTable" class="table table-striped

        table-bordered table-hover">

        <thead class="thead-dark">

          <tr>

            <th style="display:none;">AutoID</th>

            <th style="display:none;">Client</th>

            <th scope="col">name</th>

            <th scope="col">status</th>

            <th scope="col">ModifiedDate</th>

            <th scope="col">lastRunDate</th>

            <th scope="col">NextRun</th>

            <th scope="col">stepcount</th>

            <th scope="col">type</th>

            <th scope="col">lastrunstatus</th>

            <th scope="col">duration</th>

          </tr>

        </thead>

      </table>

    </div>

    <-- Script/Library blocks and external calls -->

  </body>

</html>

As you will notice in the snippet, we removed all the external calls to style sheets or scripts as they are pretty simple to understand, can be viewed in the GitHub repository https://github.com/PacktPublishing/Automating-Salesforce-Marketing-Cloud/tree/main/Chapter08, and take up a lot of unnecessary space and can cause clutter. These were replaced by HTML comments showing where they would be placed if displayed.

As previously said, there is not much there. Most of it is dynamically created through the client-side JavaScript we will go over a little later in this chapter. Mostly, you just need to have the div container, the export button, and then the table that is going to be holding your data.

The only part of the table that you need to manually set is the head. This is the listing of each of the column names you want to label your data. As you may notice in the code, we have two columns with the style of display:hidden. This is to hide them from being displayed as although they are important to have for exporting or if you need them for reference, they are not necessary to be displayed in the visual table. So, by putting this CSS on these columns, we are able to hide this info from being displayed.

As you may notice, we have some inline styling in my HTML, which in almost every way is a no-no for web development, but as the div and those th tags are the only places we need CSS outside the external style sheets, we just inlined it to have it easier to reference what it is affecting. Feel free to move this to a style sheet, especially if you start to add more custom CSS to your dashboard.

The HTML and CSS parts are pretty easy, considering we are using external libraries and plugins to do most of the work. So, next, we will move to the JavaScript section to discuss utilizing these plugins and libraries.

Client-side JavaScript
There are a ton of libraries and plugins that we are using, so this actually severely simplifies the JavaScript needed and makes it a much more readable syntax. As stated previously though, using this can lead to page bloat and slow things down, so feel free to adjust it to a more custom and efficient option instead. In fact, if you have the skill to do so, we would highly recommend it. There are a lot of opportunities here.

The first part we are going to go over about the JavaScript used in this page is the creation of the baseline for the automation URL inside of Automation Studio and setting the JavaScript var of the JSON coming from the SSJS:

      let currentUrl = document.referrer;

      let autoURL = currentUrl + 'cloud/#app/Automation

        Studio/AutomationStudioFuel3/%23Instance/';

      let myJSON = <ctrl:var name=json />;

This is pretty simple JavaScript. As this is coming from inside the Marketing Cloud UI, you can use the referrer URL and just add the context for Automation Studio to create the base link for linking to the automation inside. All you would do is append the AutoID of the automation to this link and it will take you to the automation inside of Automation Studio.

The most interesting part is the passing of the JSON from SSJS to JavaScript. In order to do this, you need to use the <ctrl> tag that is proprietary to Marketing Cloud, use the var aspect, and then provide the name of the variable. Once you do this, it will act similarly to any other personalization or inline call of an AMPscript variable, but with the SSJS variable. This means it will fill in the JSON value on the client side, allowing for the manipulation of it by the JavaScript.

buildAutoTable()
With the JSON passed from SSJS to JavaScript, we can now begin building our table. To do this, we need to create a function using the dataTable plugin. The following function is utilized to build out the table we saw in Figure 8.7:

      function buildAutoTable() {

        let autoTable = $('#autoTable').DataTable( {

             "data": myJSON,

             "paging": true,

             "pageLength": 20,

             "lengthMenu": [ [10, 20, 50, -1], [10, 20, 50,

               "All"] ],

             "search": { "caseInsensitive": false },

             "columns": [

               {"data": "AutoID",

                "visible":false

               },

               {"data": "ClientID",

                "visible":false

               },

               {"data": "Name",

                 "render": function(data, type, row, meta){

                    if(type === 'display'){

                        data = '<a href="' + autoURL +

                          row["AutoID"] + '"

                          target="_blank" >' + data +

                          '</button>';

                    }           

                    return data;

                 }

               },

               {"data": "Status"},

               {"data": "modifiedDate", "type":

                 'date-mm-dd-yyyy'},

               {"data": "LastRunDate"},

               {"data": "NextRun",

                 "render": function(data, type, row, meta){

                    if(type === 'display'){

                        data = data == '12/31/1969 06:00

                          PM' ? '' : data;

                    }           

                    return data;

                 }

               },

               {"data": "StepCount"},

               {"data": "Type"},

               {"data": "LastRunStatus"},

               {"data": "lastRunDuration"}

             ]

        } );

        return autoTable;

      }

      $(document).ready(function() {

        let autoTable = buildAutoTable();

      });

Most of this is setting parameters and values to toggle options for the output table of the dataTables output. The main point we should focus on is the columns section of this JSON config we are pushing to the DataTable method. This is where we control what columns are getting output as well as any custom manipulation and attribution of the data in those columns. For instance, you will notice in the Name column, we have a function that wraps the data point inside an a tag to link to the Automation Studio page for that automation. You will also notice for the first two columns, like in the HTML, we have these as not visible to hide them from being displayed.

Rather than providing a lot of information on how to use dataTables and set up the config JSON, we instead recommend going to the documentation for more information on how and why this is set up in this manner and other options on what can be done.

Lastly, you will notice a jQuery statement, stating that when the page is loaded we should run a function to run the buildAutoTable() function. Next, we are going to go over the JavaScript function utilized to allow you to export the Auto Dashboard to a CSV file that you can view in Excel.

download()
This function takes the JSON data we assigned to the myJSON variable coming from the SSJS and then transforms it into a CSV file and then downloads it to your local machine. The major benefit here is that it can allow you to create snapshots in time of what your automations' statuses and runtimes were as well as giving you something that you can view offline, and you do not need to be logged in to Marketing Cloud to view it.

Take a look at the following snippet to see the function we are referencing:

      //Export CSV JS

      function download() {

        let csv = '';

        let items = myJSON;

        // Loop the array of objects

        for(let i = 0; i < items.length; i++){

            let numKeys = Object.keys(items[i]).length

            let counter = 0

            // If this is the first row, generate the

            // headings

            if(i === 0){

               // Loop each property of the object

               for(let key in items[i]){

         // This is to not add a comma at the last cell

         // The '\r\n' adds a new line

                   csv += key + (counter+1 < numKeys ? ','

                     : '\r\n' )

                   counter++

               }

            }else{

               for(let key in items[i]){

                   let data = items[i][key];

                   data = data == '12/31/1969 06:00 PM' ?

                     '' : data;

                   csv += data + (counter+1 < numKeys ? ','

                     : '\r\n' )

                   counter++

               }

            }

        }

        // Once we are done looping, download the .csv by

        // creating a link

        let link = document.createElement('a')

        link.id = 'download-csv'

        link.setAttribute('href',

          'data:text/csv;charset=utf-8,' +

           encodeURIComponent(csv));

        link.setAttribute('download',

          'AutoDashExport.csv');

        document.body.appendChild(link)

        document.querySelector('#download-csv').click()

      }

All in all, it is not really that complex of a function. It utilizes JavaScript to transform the JSON into a rowset that is separated by commas and columns and uses newline characters to separate rows. It then builds out an a tag that has the href value set to download the file that we just created and then uses JavaScript to programmatically click this new link and initiate the download.

That is the whole code and explanation, so now all you need to do is copy/paste the code from GitHub into the CloudPages you set up previously, then click Save and Publish. Once you do that, you can then go to the top navigation bar and select the AppExchange option. See the following screenshot for an example:



Once you hover over that, there will be a dropdown that should include the name of the custom application that you created. All you need to do there is just click on the name and it will load in a new iframe in your Marketing Cloud instance. That's it! You now have a fully functional Automation Dashboard inside your account and all the knowledge to build your own custom web applications to be used inside Marketing Cloud.
