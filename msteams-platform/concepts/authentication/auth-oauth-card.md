---
title: Using the OAuth card for authentication
description: Describes Using the OAuth card for authentication
keywords: teams authentication
ms.date: 07/02/2018
---
# Using Azure Bot Service for Authentication in Teams

## Overview

Traditionally, the work to authenticate users through a bot has been complex. It’s a full-stack challenge that involves building a web experience, integrating with external OAuth providers, token management, and handling the right server-to-server API calls to ensure the authentication flow is completed in a secure manner. Users also end up with clunky experiences with entering in “magic numbers”.

With Azure Bot Service’s new OAuthCard, it is now easier than ever for your Teams bot to sign your users in and access external data providers. Whether you’ve already implemented auth and you want to switch over to something simpler, or if you are looking to add authentication to your bot service for the first time, this new system can make it easier.

## How does the Azure Bot Service help me do auth?

Full documentation is available here.

## Main benefits for Teams developers

* Provides an out-of-box web-based authentication flow: you no longer have to write and host a web page to directs to external login experiences, or provide a redirect
* Seamless for end users: complete the full signin experience right within Teams
* Easy token management: you no longer have to implement your own token storage system – instead, the Bot Service takes care of token caching and provides a secure mechanism for fetching those tokens
* Complete SDKs: easy to integrate and consume from your bot service
* Out-of-box support for many popular OAuth providers, such as AAD/MSA, Facebook, and Google

## When should I implement my own solution?

Because access tokens are sensitive information, you may not wish to have them stored in an external service. In this case, you may choose to still implement your own token management system and login experience within Teams, as described here.

## Getting started with OAuthCard in Teams

You’ll first need to configure your Azure bot service to set up external authentication providers. Read the tutorial for detailed steps.

To enable auth using the Bot Service, you need to make these basic additions to your code: 
Include token.botframework.com in the validDomains section of your app manifest. This is needed because Teams will embed the Bot Service’s login page.

Whenever your bot needs to access authenticated resources, fetch the token from the Bot Service. If no token is available, send a message with an OAuthCard to the user requesting them to log into the external service.

Handle the login completion activity – this ensures that the authentication request and the token are associated with the user currently interacting with your bot. 
Retrieve the token whenever your bot needs to perform authenticated actions, such as calling external REST APIs.

In your dialog code, you’ll need to add this snippet (C#), which checks for an existing access token:

// First ask Bot Service if it already has a token for this user
var token = await context.GetUserTokenAsync(ConnectionName).ConfigureAwait(false);
if (token != null)
{
    // use the token to do exciting things!
}
else
{
    // If Bot Service does not have a token, send an OAuth card to sign in 
    await SendOAuthCardAsync(context, (Activity)context.Activity); 
} 
If one doesn’t exist, your code will then send a message with an OAuthCard to the user: 
private async Task SendOAuthCardAsync(IDialogContext context, Activity activity) 
{ 
    await context.PostAsync($"To do this, you'll first need to sign in."); 
 
    var reply = await context.Activity.CreateOAuthReplyAsync(_connectionName, _signInMessage, _buttonLabel).ConfigureAwait(false); 
    await context.PostAsync(reply); 
 
    context.Wait(WaitForToken); 
} 
To handle the login complete activity, you’ll need to process this Invoke: 
if (activity.Name == "signin/verifyState") 
{ 
  // We do this so that we can pass handling to the right logic in the dialog. You can 
  // set this to be whatever string you want. 
  activity.Text = "loginComplete"; 
  await Conversation.SendAsync(activity, () => new Dialogs.RootDialog()); 
 
  return Request.CreateResponse(HttpStatusCode.OK); 
} 
In your dialog code, you can then retrieve the token from the Bot auth service: 
if (text.Contains("loginComplete")) 
  { 
    // Handle login completion event. 
    JObject ctx = activity.Value as JObject; 
 
    if (ctx != null) 
    { 
      string code = ctx["state"].ToString(); 
 
      var oauthClient = activity.GetOAuthClient(); 
      var token = await oauthClient.OAuthApi.GetUserTokenAsync(activity.From.Id, ConnectionName, magicCode: code).ConfigureAwait(false); 
      if (token != null) 
      { 
        // Make whatever API calls here you want 
        await context.PostAsync($"Success! You are now signed in."); 
      } 
    } 
  // Need to respond to the Invoke. 
  return; 
}

## Support on mobile

As of July 2018, using the Bot Service to authenticate is not supported on mobile. For now, sign-in is limited to web/desktop Teams clients.

## Using OAuthCard with messaging extensions

You can also use Azure Bot Service to connect third-party providers to your messaging extension. The flow is the same as with a bot, except instead of returning an OAuthCard, your service will return a login prompt.

The following snippet (C#) illustrates how to craft the login response: 

var token = await client.OAuthApi.GetUserTokenAsync(activity.From.Id, ConnectionName).ConfigureAwait(false);
 
if (token == null) 
{ 
  // Send the login response with the auth link. 
  string link = await client.OAuthApi.GetSignInLinkAsync(activity, ConnectionName); 
 
  response = new ComposeExtensionResponse() 
  { 
    ComposeExtension = new ComposeExtensionResult() 
  }; 
  response.ComposeExtension.Type = "auth"; 
  response.ComposeExtension.SuggestedActions = new ComposeExtensionSuggestedAction() 
  { 
    Actions = new List<CardAction>() 
      { 
        new CardAction(ActionTypes.OpenUrl, title: "Sign into this app", value: link) 
      } 
    }; 
  return response; 
} 

Note in the example above that you need to make the call to GetSignInLinkAsync directly against the client.OAuthApi property.

When the user successfully completes the login sequence, your service will receive another Invoke request containing the original user query, along with a state parameter containing the “magic code”. You can now fetch the token using this string, along with the user ID and connection name. 
var query = activity.GetComposeExtensionQueryData(); 
JObject data = activity.Value as JObject; 
 
var client = activity.GetOAuthClient(); 
 
// Check if the request comes with login state 
if (data != null && data["state"] != null) 
{ 
  var token = await client.OAuthApi.GetUserTokenAsync(activity.From.Id, ConnectionName, data["state"].ToString()); 
 
  // Do stuff with the token here. 
 
} 