It all started when I was researching for a report to pull large data sets on the Usage of WHFB in an organization. Fortunately, I found this great blog [here](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/azure-ad-sign-in-logs-workbooks-know-who-is-using-windows-hello/ba-p/2661980) by Michael Hildebrand which helped me to kick start. 

However, I had a question about "How many users have registered for WHFB Vs their Usage?" 

As you know, the User registration details for WHFB(or any method registered) are available in our [Azure AD Authentication Methods Activity reporting](https://docs.microsoft.com/en-us/azure/active-directory/authentication/howto-authentication-methods-activity) in the Azure AD Portal at granular level.

<img width="732" alt="image" src="https://user-images.githubusercontent.com/111733151/185876360-183ba8fc-597b-41db-9c12-12c866048f09.png">



So, I was also looking to retrieve the User Registration Details in larger datasets and compare it with the Usage for the last 30 days (max). 

This post will take you through the steps of retrieving the User Registration Details for WHFB, send to a Custom Log in the Log Analytics Workspace and then visualize the Usage data with respect to the Registration Details.

Let's get started!!!

You likely already have the pre-reqs for this, especially if you’re doing much w/ Azure AD today.    
	• [Azure Monitor workbooks for reports | Microsoft Docs](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/howto-use-azure-monitor-workbooks)

Firstly, let's retrieve the User Registration Details for WHFB to a Custom Log in Log Analytics workspace. For which, I am using a Logic App with the below logic considering all the [Service limits](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-limits-and-config?tabs=azure-portal) while fetching large datasets too.

When you run a call to the Graph API that has more results than your $top count (is limited to max 999) then you get a property in the body of the request called @odata.nextLink. If the number of results is less than or equal to your top count then that property simply doesn’t exist. 

<img width="590" alt="image" src="https://user-images.githubusercontent.com/111733151/185876510-a1e734c1-dde5-4716-a164-f46541c31061.png">




There are often many ways to achieve an outcome, and here’s how I’ve done it. I will be handling the pagination of this Graph API, or indeed any other OData compliant API, through a HTTP Connector in the Logic Apps.



I initialized a few variables: A Boolean that is used in the looping logic, an array variable to store the items retrieved from the API, and a string variable for the odatanext link.


 

![image](https://user-images.githubusercontent.com/111733151/185876649-31da0591-18eb-4ca6-8470-907e7414c049.png)


GET the Registered WHFB Users through the HTTP Request. Since, I have total registered Users less than 999 and to handle odatanextlink, I will be using $top = 10.
The default top count is 100 and you can cut down on the time this processes by making it as high as it goes: 999 if you have large number of users.

In order to use Azure ActiveDirectory Oauth for the HTTP GET request, you need to register an application on Azure AD, create a Client secret and also add API permissions for AuditLog.Read.All and UserAuthenticationMethod.Read.All



![image](https://user-images.githubusercontent.com/111733151/185959639-26c4a535-635f-4613-91f1-913a798ce6f3.png)


![image](https://user-images.githubusercontent.com/111733151/185876768-8cfd4eeb-464a-486a-95e3-9cc737dd07a8.png)


Now Parse the JSON and then send the 1st Iteration Data to log Analytics Custom Log.

![image](https://user-images.githubusercontent.com/111733151/185876812-d696ed5f-4f26-4f4a-8261-6cdfa6d9f372.png)


Now add all the items to the array variable you initialized earlier. The input for the For each is body(‘GetRegisteredUsers’)?[‘value’]

![image](https://user-images.githubusercontent.com/111733151/185876871-8b5dfa96-eb40-4846-9676-764d711466c8.png)


Now we see if there’s a next page by crudely looking for the string ‘odata.nextLink’ in the entire response from the HTTP GET. Hopefully nobody creates a user account with that exact string in any of the attributes we’ve picked out with the $select query! :P 

![image](https://user-images.githubusercontent.com/111733151/185876945-9ded810b-8c6c-4745-885e-afa9e88d513f.png)


In the If false/No branch, do nothing. In the if true/Yes branch, decode the @odata.nextLink property in a Set variable action. The variable being the string that was initialized earlier. The expression is decodeUriComponent(body(‘GetRegisteredUsers’)?[‘@odata.nextLink’])


![image](https://user-images.githubusercontent.com/111733151/185876980-d783fb38-62d3-49e6-ad58-9cbb2dacbf2d.png)


Now add a Do until, or Until loop with the Boolean variable is equal to true( here, we are still under the IF TRUE condition). Inside, that the first action is another HTTP GET with the same OAuth parameters but the URI is the string variable (the URI decoded @odata.nextLink property). All the original queries are included in the nextLink so no need to include them again:

![image](https://user-images.githubusercontent.com/111733151/185877035-3ed76291-53f7-4997-9cb6-60cd0107585e.png)


Another loop to Parse the JSON and send the data to Log analytics for the Current Item.

![image](https://user-images.githubusercontent.com/111733151/185877096-4e39e148-4158-472a-a388-96c55b796a81.png)


Now another condition. We’re still within the Do Until here. The expression within the Set variable action (in the If true branch) called update odatanextLink is decodeUriComponent(body(‘HTTP’)?[‘@odata.nextLink’])


![image](https://user-images.githubusercontent.com/111733151/185877148-eeb39f0d-eda7-4447-9e7b-eb10a057072c.png)



It will take some time to send the data to Log Analytics Workspace's Custom Log depending on the number of users. You can configure the Service Limits for Until Loop depending on the number of users according to the limits described here : https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-limits-and-config?tabs=azure-portal#until-loop

Once the data is moved to the Custom Log, we would see it as below

![image](https://user-images.githubusercontent.com/111733151/185877209-49c549f6-8611-4548-8640-a8b17f8f92ff.png)


Now, we can use this Registration details from the Custom Log, Compare with our WHFB Usage from Sign in Logs and generate the Analysis. The JSON code for this specific part can be used and build on top of [Michael's workbook](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/azure-ad-sign-in-logs-workbooks-know-who-is-using-windows-hello/ba-p/2661980)

This JSON code will include 

Section 1 - “Windows Hello for Business Registration Vs Usage”
A pie chart showing number of users registered with WHFB and number of users using WHFB among them.

Section 2 - “Windows Hello for Business Usage – Per-Device and Per-User Authentication Counts”
A table showing each device, each user and the counts of times the user signed-in via WH4B along with their User Prinicpal Name.


![image](https://user-images.githubusercontent.com/111733151/185877283-16e73359-81c8-4ee9-872e-5a276b51d7d8.png)

