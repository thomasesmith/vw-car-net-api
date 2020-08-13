# Connecting to the Volkswagen Car-Net API
### The Update!

This document is intended to be a report of my findings experimenting with the VW Car-Net app's api since the service's update in October of 2019. How you use this information is your responsibility. **I have no idea if using the API in this manner violates the terms of service of the VW Car-Net service that you agreed to when you signed up. So, [if I were us](https://www.youtube.com/watch?v=pvCQvIrH35s), I would just assume that it does. Proceed at your own risk.**.

This README will stay code-agnostic and just detail the http requests to the API, what is required to include with each request, and a general idea of what you can expect back from the server in response. 


***

## First, You Need Access Tokens
After the update in October of 2019, Car-Net now uses an Oauth scheme to log you in to the app. So in order to get tokens programmatically, you have to scrape some values from the log in forms that the process generates. It's a bit of a hack, and you'll have to perform four separate requests before you can even request access tokens, but it does work. Don't worry, we'll go in to great detail for each of them. 

But before we do, a quick word about how PKCE works...
 
#### "Code Challenge" / "Code Verifier"

Oauth PKCE schemes like the one Car-Net uses require you to submit a `code_challenge` string in the first request that generates the log in form, and then in the final step submit a `code_verifier` string that relates cryptographically to that original `code_challenge` string you submitted. There are countless better ways to learn how Oauth works than this document, and you should find them and read them, but if you don't want to go to that trouble, you can also just use [this handy little PKCE generator](https://tonyxu-io.github.io/pkce-generator) by [Tony Xu](https://tonyxu.io) that will generate a Code Challenge string out of a submitted Code Verifier.

Simply enter *a 64-character random hex value* (characters a-f and 0-9) in to the "Verifier" field, click "Generate Code Challenge," and there you go. 

However it is you want to generate a valid PKCE pair of values, **save both values somewhere for later**. You're going to need them. Let's get going. 

### Auth, Step 1

For the very first of the auth requests, we'll essentially be asking VW's servers to generate a log-in form for us, then snagging values out of the markup to handle the subsequent requests. Start with:

#### Request
```
GET https://b-h-s.spr.us00.p.con-veh.net/oidc/v1/authorize?redirect_uri=car-net%3A%2F%2F%2Foauth-callback&scope=openid&prompt=login&code_challenge=[CODE CHALLENGE]%3D&state=[RANDOMLY GENERATED UUID VALUE]&response_type=code&client_id=2dae49f6-830b-4180-9af9-59dd0d060916%40apps_vw-dilab_com
```

There are 5 query values in that URL. The only two you need to worry about are the `state` value where you must submit any old random UUID value, and the `code_challenge` field, where you must submit the code challenge from the PKCE pair you generated before (leave the `%3D` at the end of this value, like it is in the example). 

> You can use your favorite search engine to read more about what a 'UUID' value is and how mocking one will satisfy the api, or just use [a UUID generator](https://www.uuidgenerator.net/version4) to make one.

> Also, in case you're wondering `2dae49f6-830b-4180-9af9-59dd0d060916%40apps_vw-dilab_com` is I believe the oauth client id of the iOS app. At least it is for the version on my phone. Either way, **copy this too**, you'll need it for a few more requests. 

#### Response
The response will be a series of `30x` redirects, one after another. Follow each of them (or instruct whatever programmatic means you're using to do so). It will eventually land on a VW branded log-in page asking you to submit your email address to begin the log in process. From this page's markup you will need to scrape some more values. Look for the form and find the chunks that look like this: 

```
...
<form class="content" id="emailPasswordForm"
      name="emailPasswordForm"
      method="POST"
      novalidate="true"
      action="/signin-service/v1/b680e751-7e1f-4008-8cc1-3a528183d215@apps_vw-dilab_com/login/identifier">

...

<input type="hidden" id="csrf" name="_csrf" value="4e9f7522-c2e7-4e6d-87a1-f87b95c6e79b"/>
<input type="hidden" id="input_relayState" name="relayState" value="8510a5c146130ec2cc31721e0fa889bd6c613607"/>
<input type="hidden" id="hmac" name="hmac" value="7ecf096d1176c00a9ffd8840190a9ab304644078015354efbc087f29d6c9ce7c"/>
...
```

**Copy the values** for `_csrf`, `relayState`, and `hmac` for use in upcoming requests. Also copy the form's `action` value.

### Auth, Step 2

The second request submits your email address. Make a POST request to the uri you copied from the `action` value in the markup of the last response. Then include a body of some form values.  

#### Request
``` 
POST /signin-service/v1/b680e751-7e1f-4008-8cc1-3a528183d215@apps_vw-dilab_com/login/identifier 
content-type: application/x-www-form-urlencoded

_csrf=[CSRF]&relayState=[RELAY STATE]&email=[YOUR CAR NET EMAIL ADDRESS]&hmac=[HMAC]
```
For this one, we have to include 4 form values in the request. `_csrf`, `relayState`, and `hmac` should equal the values we just copied from the last request's response, but the `email` value should be the set to whatever email address you use to log in to your Car-Net account. 

#### Response 
Again, this will result in a series of `30x` redirects, but if you follow them all, it will finally settle on the form where you would be submitting your password. You again need to scrape some values from the markup of this response and grab one more value. Find this: 

```
...
<form class="content" id="credentialsForm"
      name="credentialsForm"
      method="POST"
      action="/signin-service/v1/b680e751-7e1f-4008-8cc1-3a528183d215@apps_vw-dilab_com/login/authenticate">
...
<input type="hidden" id="hmac" name="hmac" value="0b6b6bd01b29c502cd31a204c6d4a2e07fe161e7ba9a3b97cec6b59c261a52d9"/>
...
```
The form generated a new `hmac` value that we must use when we submit our password in the next request. Also **copy again** this form elements `action` value, since the next request will be made to that uri. 

### Auth, Step 3
This third request is the submission of both your password and email address (don't worry, your password is being sent along just as secure lines as it would be if you used the official app itself to submit it).
#### Request
```
POST https://identity.na.vwgroup.io/signin-service/v1/b680e751-7e1f-4008-8cc1-3a528183d215@apps_vw-dilab_com/login/authenticate
content-type: application/x-www-form-urlencoded

_csrf=[CSRF]&relayState=[RELAY STATE]&email=[CAR NET EMAIL ADDRESS]&hmac=[LAST HMAC COPIED]&password=[CAR NET PASSWORD]
```
We include 5 form values in this request. `_csrf`, `relayState` should equal the values you copied from the response of Auth Step 1, and `hmac` should equal the value you copied from the response of Auth Step 2, the `email` value should be whatever email address you use to log in to your Car-Net account, and the `password` should of course be the password you use to log in to you Car-Net account 

#### Response
You guessed it, it responds with another series of `30x` redirects. Only this time, they will eventually land you on a url you can't load, a `car-net://` uri. This is the point at which the app itself would normally take over the process. So in the event of a successful log in, in the details of the final `302` redirect, you will find something that looks like this uri: 
 ```
HTTP/2 302 
location: car-net:///oauth-callback?state=d046b114-7aa8-46d5-84e3-938fcd64b482&code=0ae3dcb147ff2c12531bdc88595b0b66eb28a2c3a338c35e6d28e6cd32516defdaf4e581bdab4f51a80cd85e88a8034
```
Now, you must copy the value of `code` so we can use it in the final auth step.

### Auth, Step 4
Finally, this request will actually result in us getting access tokens! 

> Take note of the change in domain!

#### Request
```
POST https://b-h-s.spr.us00.p.con-veh.net/oidc/v1/token HTTP/2
content-type: application/x-www-form-urlencoded
accept: */*
accept-encoding: gzip, deflate, br
user-agent: Car-Net/60 CFNetwork/1121.2.2 Darwin/19.3.0
accept-language: en-us

grant_type=authorization_code&code=[CODE VALUE]&client_id=2dae49f6-830b-4180-9af9-59dd0d060916%40apps_vw-dilab_com&redirect_uri=car-net%3A%2F%2F%2Foauth-callback&code_verifier=[CODE VERIFIER VALUE THAT YOU GENERATED EARLIER]
```
#### Response
```
{
  "access_token": "eyJraWQiOiJNenZDSkJHNVFh...",
  "expires_in": 1800,
  "id_token": "eyJraWQiOiJ1R0YzNm9LWGt1Vms4...",
  "id_expires_in": 1800,
  "token_type": "Bearer",
  "refresh_token": "eyJraWQiOiJNenZDSkJHNVF...",
  "refresh_expires_in": 31536000
}
```
> If this returns anything other than a 200 status code, it is probably due to an invalid `code_verifier`. Try re-generating a PKCE pair and going again.  

Finally! We have some tokens. Let's see what we can do with them...

***

## First, Get Your User Profile and List of Vehicles
This is a simple GET request using the `id_token` you just received right in the uri, and using the `access_token` in the `authorization` header.
> Don't forget to include "Bearer " before your access token!

#### Request

```
GET https://b-h-s.spr.us00.p.con-veh.net/account/v1/enrollment/status?idToken=[ID TOKEN]
authorization: Bearer [ACCESS TOKEN]
```
#### Response 
You will get back a lot of information about yourself and your status with Car-Net. So much that it might spook you: your birth date, the  IMEI number of the cell device in your car, `stolenFlag`?! 
```
{
  "data": {
    ...
    "customer": {
      "userId": "4568c4f9-6e5b-4308-99f7-eb276ba0fa26",
      ...
      "vwId": "c65512a4-9c58-4fd6-a42a-549cc0314de6vwcarnetportal-prod-002-appserver-000",
       ...
      "firstName": "Paul",
      "lastName": "Newman",
      "preferredLanguage": "English",
      "birthDate": ...,
      "address": {
        ...
      },
      "phones": [
        {
          ...
        }
      ],
      "emails": [
        {
          ...
        }
      ],
      "updatedAt": 1574754533000
    },
    "vehicleEnrollmentStatus": [
      {
        "vehicleId": "abcdef12-3456-7890-abcd-ef1234567890",
        "carName": {},
        "associationStatus": "ASSOCIATION_VERIFIED",
        "vehicle": {
          "vin": "...",
          "vehicleId": "abcdef12-3456-7890-abcd-ef1234567890",
          "brand": "VW",
          "modelName": "e-Golf",
          "modelYear": "2016",
          "modelCode": "BE12B1",
          "modelDesc": "egolf_2016",
          "stolenFlag": "N",
          "tspProvider": "VZT",
          "ocuSim": {
            "imei": ...,
            "defaultMno": "Verizon"
          },
          ...
        },
        "rolesAndRights": {
          "vehicleId": "...",
          "userId": "...",
          ...
          "tspAccountNum": "12345678",
          "privileges": [
            ...
          ],
          ...
        },
        ...
      }
    ]
  }
}
```
You will get a lot back from this one (I would encourage you to experiment), but the values in this request that you'll definitely need for subsequent requests are your `userId`, your `vwId`, and the `vehicleId`, `tsp`, and `tspAccountNum` values that are associated with the vehicle you want to control. 

## Now You Can Get A Vehicle's Status Summary
Now you can begin poking around.  Start by getting your vehicle's current "status." This is a summary of your vehicle that includes a lot of stuff: your car's GPS coordinates (if applicable), its odometer reading, door lock status, its battery/charge status, approximate remaining range, etc.

#### Request
```
GET https://b-h-s.spr.us00.p.con-veh.net/rvs/v1/vehicle/[VEHICLE ID] HTTP/2
authorization: Bearer [ACCESS TOKEN]
```
#### Response
```
{
  "data": {
    "currentMileage": ...,
    "nextMaintenanceMilestone": {
      "mileageInterval": 0,
      "absoluteMileage": ...
    },
    "timestamp": ...,
    "exteriorStatus": {
      "secure": "SECURE",
      "doorStatus": {
        "frontLeft": "CLOSED",
        "frontRight": "CLOSED",
        "rearLeft": "CLOSED",
        "rearRight": "CLOSED",
        "trunk": "CLOSED",
        "hood": "CLOSED",
        ...
      },
      "doorLockStatus": {
        "frontLeft": "LOCKED",
        "frontRight": "LOCKED",
        "rearLeft": "LOCKED",
        "rearRight": "LOCKED"
      },
      "windowStatus": {
        "frontLeft": "CLOSED",
        "frontRight": "CLOSED",
        "rearLeft": "CLOSED",
        "rearRight": "CLOSED"
        ...
      },
      "lightStatus": {
       ...
      }
    },
    "powerStatus": {
      "cruiseRange": 148,
      "fuelPercentRemaining": 0,
      "cruiseRangeUnits": "KM",
      "cruiseRangeFirst": 148,
      "battery": {
        "chargePercentRemaining": 100,
        "minutesUntilFullCharge": 15,
        "chargePlug": "PLUGGEDIN",
        "triggeredByTimer": "false"
      }
    },
    "location": {
      "latitude": ...,
      "longitude": ...
    },
    "lastParkedLocation": {
      "latitude": ...,
      "longitude": ...
    },
    ...
    "lockStatus": "LOCKED"
  }
}
```

## Refresh Your Tokens When They Need It
After 30 minutes, your `access_token` and `id_token` will expire. You will need to use your `refresh_token` to request a new set of tokens that will last another 30 minutes. Here's how to:

#### Request
```
POST https://b-h-s.spr.us00.p.con-veh.net/oidc/v1/token HTTP/2
content-type: application/x-www-form-urlencoded
accept: */*

code_verifier=[CODE VERIFIER YOU GENERATED EARLIER]&grant_type=refresh_token&client_id=2dae49f6-830b-4180-9af9-59dd0d060916%40apps_vw-dilab_com&refresh_token=[REFRESH TOKEN]
```
#### Response
```
{
  "access_token": "eyJraW56ryjyukZDSkJHNVFh...",
  "expires_in": 1800,
  "id_token": "eyJraWQiOiJ1R0676y9LWGt1Vms4...",
  "id_expires_in": 1800,
  "token_type": "Bearer",
  "refresh_token": "eyJraWQiOiJ78lhkuyJHNVF...",
  "refresh_expires_in": 31536000
}
```
You should get back a fresh new set of tokens. If you just keep executing this refresh request before your `refresh_token` expires, you can avoid doing that obnoxious log in process for quite some time.

So, this is all good for _reading_ information, but what if you want to tell your car to do things? For that we need to request one more token.

## Get a "TSP Token" to Command The Vehicle
To make commands of a vehicle, you need to request what the api refers to as a TSP Token. This is also where your Car-Net PIN comes in to play.

#### Request
```
POST https://b-h-s.spr.us00.p.con-veh.net/ss/v1/user/[USER ID]/vehicle/[VEHICLE ID]/session HTTP/2
> content-type: application/json;charset=UTF-8
> authorization: Bearer [ACCESS TOKEN]

{
  "accountNumber": "[TSP ACCOUNT NUMBER]",
  "idToken": "[ID TOKEN]",
  "tspPin": "[YOUR CAR-NET PIN]",
  "tsp": "[TSP]"
}
```

#### Response
This returns a lot, some of which you may find interesting, but what we're looking for is this value:

```
{
    "data": {
        ...
        "tspToken": "3c777373653a536563757269747920786d6c3a777..."
    }
}
```

You will need this TSP Token for every request after this.

> For some reason this token doesn't come with expiration information, but I have found that after about 45 minutes, this token needs to be refreshed. In order to refresh the `tspToken`, just run this same request again (of course with a valid and fresh `access_token`/`id_token` set) and you will get back a new one.

## Force Refresh of Vehicle Status/Summary

Sometimes the vehicle status that we talked about above gets a little stale (as evident by an old `timestamp` value), but this tends to be when nothing about the car is changing much. For instance, while charging my eGolf, the status response updates regularly and doesn't require a forced refresh, but when the car is parked in a way that nothing about the car is really changing, it lets the status response get a little old. 

If you want to force the system to re-poll the vehicle and get an updated summary, perform the following... 

#### Request
PUT a json body like the following (and don't forget these important headers) to refresh the status of the vehicle.

```
PUT https://b-h-s.spr.us00.p.con-veh.net/mps/v1/vehicles/[ACCOUNT NUMBER]/status/fresh HTTP/2
content-type: application/json;charset=UTF-8
x-user-id: [USER ID]
x-user-agent: mobile-ios
x-app-uuid: [ANY RANDOM UUID VALUE]
authorization: Bearer [ACCESS TOKEN]

{
  "tsp_token": "[TSP TOKEN]",
  "email": "[YOUR CAR NET EMAIL ADDRESS]",
  "vw_id": "[YOUR VW ID]"
}
```

#### Response

```
{
  "data": {
    "correlationId": "...",
    "command": "fetch_vehicle_status",
    "result": 0
  }
}
```
> Just a note about most of the command requests you can make: getting a response back doesn't mean your request has already been fulfilled in the real world. It just spits back an affirmative response to it. If you were to run this request, then request the status of the car immediately after, you will not find that the status response now indicates the vehicle's doors are now locked. It takes me about 20 seconds for that to actually reflect. You might notice that the API itself responds slowly to PUT and PATCH command requests (they take sometimes 10 seconds to respond), it is my belief that this is due to the api reaching out to the vehicle's cellular radio, and the upstream latency inherit with that.


## Locking/Unlocking The Doors
#### Request
```
PUT https://b-h-s.spr.us00.p.con-veh.net/mps/v1/vehicles/[ACCOUNT NUMBER]/status/exterior/doors HTTP/2
content-type: application/json;charset=UTF-8
x-user-id: [USER ID]
x-user-agent: mobile-ios
x-app-uuid: [ANY RANDOM UUID VALUE]
authorization: Bearer [ACCESS TOKEN]

{
  "tsp_token": "[TSP TOKEN]",
  "lock": true,
  "email": "[YOUR CAR NET EMAIL ADDRESS]",
  "vw_id": "[YOUR VW ID]"
}
```
#### Response
````
{
  "data": {
    "correlationId": "...",
    "command": "lock_doors",
    "result": 0
  }
}
````
And then to unlock...
#### Request
```
PUT https://b-h-s.spr.us00.p.con-veh.net/mps/v1/vehicles/[ACCOUNT NUMBER]/status/exterior/doors HTTP/2
content-type: application/json;charset=UTF-8
x-user-id: [USER ID]
x-user-agent: mobile-ios
x-app-uuid: [ANY RANDOM UUID VALUE]
authorization: Bearer [ACCESS TOKEN]

{
  "tsp_token": "[TSP TOKEN]",
  "lock": false,
  "email": "[YOUR CAR NET EMAIL ADDRESS]",
  "vw_id": "[YOUR VW ID]"
}
```
#### Response
```
{
  "data": {
    "correlationId": "...",
    "command": "unlock_doors",
    "result": 0
  }
}
```
## Flash Headlights/Honk Horn
#### Request
```
PUT https://b-h-s.spr.us00.p.con-veh.net/mps/v1/vehicles/[ACCOUNT NUMBER]/status/exterior/horn_and_lights HTTP/2
content-type: application/json;charset=UTF-8
x-user-id: [USER ID]
x-user-agent: mobile-ios
x-app-uuid: [ANY RANDOM UUID VALUE]
authorization: Bearer [ACCESS TOKEN]

{
  "email": "[EMAIL ADDRESS]",
  "horn": true,
  "lights": true,
  "tsp_token": "[TSP TOKEN]",
  "vw_id": "[VW ID]"
}
```
#### Response
```
{
  "data": {
    "correlationId": "...",
    "command": "flash_lights_and_honk_horn",
    "result": 0
  }
}
```
>You can also **just** honk the horn, by changing the JSON to `"lights": false`, or you can **just** flash the lights by changing the JSON to `"horn": false`.

## Start/Stop Charging
#### Request
```
PATCH https://b-h-s.spr.us00.p.con-veh.net/mps/v1/vehicles/[ACCOUNT NUMBER]/status/charging HTTP/2
content-type: application/json;charset=UTF-8
x-user-id: [USER ID]
x-user-agent: mobile-ios
x-app-uuid: [RANDOM UUID]
authorization: Bearer [ACCESS TOKEN]

{
  "active": true,
  "email": "[EMAIL ADDRESS]",
  "tsp_token": "[TSP TOKEN]",
  "vw_id": "[VW ID]"
}
```
> Watch out! Unlike the others, charging related requests are of method `PATCH`.

### Response

```
{
  "value": {
    "transaction_id": "...",
    "command": "start_charge"
  }
}
```
> To STOP charging, simply change the JSON value of `active` from true to false.

## Setting the Maximum Charging Current
Sometimes you don't want your car to pull all the amps it can from a charger. Here's how to set it to max out that amperage, or how to instruct your vehicle not to use draw at its maximum ability. 
#### Request 
```
PUT https://b-h-s.spr.us00.p.con-veh.net/mps/v1/vehicles/[ACCOUNT NUMBER]/settings/max_charge_current HTTP/2
content-type: application/json;charset=UTF-8
x-user-id: [USER ID]
x-user-agent: mobile-ios
x-app-uuid: [RANDOM UUID]
authorization: Bearer [ACCESS TOKEN]

{
  "max_charge_current": "max",
  "email": "[EMAIL ADDRESS]",
  "tsp_token": "[TSP TOKEN]",
  "vw_id": "[VW ID]"
}
```
> The values that the API will accept for `max_charge_current` probably differ depending on your vehicle, but I happen to know that for the 2016 eGolf SE (w/ the optional DC Fast Charging package) the acceptable values of `max_charge_current` are int values 5, 10, 13, and a string value of "max". 

#### Response
```
{
  "data": {
    "correlationId": "...",
    "command": "set_max_charge_current",
    "result": 0
  }
}
```
## Retrieve Your Vehicle's Climate System Settings
#### Request
```
PUT https://b-h-s.spr.us00.p.con-veh.net/mps/v1/vehicles/[ACCOUNT NUMBER]/status/climate/details HTTP/2
content-type: application/json;charset=UTF-8
x-user-id: [USER ID]
x-user-agent: mobile-ios
x-app-uuid: [ANY RANDOM UUID VALUE]
authorization: Bearer [ACCESS TOKEN]

{
  "tsp_token": "[TSP TOKEN]",
  "email": "[YOUR CAR NET EMAIL ADDRESS]",
  "vw_id": "[YOUR VW ID]"
}
```
#### Response
```
{
  "data": {
    "outdoor_temperature": 74,
    "target_temperature": 77,
    "climate_on": false,
    "defrost_on": false,
    "unplugged_climate_control_enabled": true,
    "triggered_by_timer": false
  }
}
```

## Turning On/Off Defroster
If your vehicle is plugged in, or if you have `unplugged_climate_control_enabled` set to `true`, you can remotely turn off or on the defroster. 
#### Request
```
PUT https://b-h-s.spr.us00.p.con-veh.net/mps/v1/vehicles/46319909/status/defrost HTTP/2
content-type: application/json;charset=UTF-8
x-user-id: [USER ID]
x-user-agent: mobile-ios
x-app-uuid: [RANDOM UUID]
authorization: Bearer [ACCESS TOKEN]

{
  "active": true,
  "email": "[EMAIL]",
  "tsp_token": "[TSP TOKEN]",
  "vw_id": "[VW ID]"
}
```

#### Response
```
{
  "data": {
    "correlationId": "...",
    "command": "start_defrost",
    "result": 0
  }
}
```
> Issue the same request but with `active` set to boolean false to **turn off** the defroster. 

## Start/Stop Climate System
If your vehicle is plugged in, or if you have `unplugged_climate_control_enabled` set to `true`, you can adjust the climate settings of your vehicle with the api. Here's how: 

#### Request
```
PUT https://b-h-s.spr.us00.p.con-veh.net/mps/v1/vehicles/[ACCOUNT NUMBER]/status/climate HTTP/2
content-type: application/json;charset=UTF-8
x-user-id: [USER ID]
x-user-agent: mobile-ios
x-app-uuid: [RANDOM UUID]
authorization: Bearer [ACCESS TOKEN]

{
  "active": false,
  "target_temperature": 77,
  "email": "[EMAIL ADDRESS]",
  "tsp_token": "[TSP TOKEN]",
  "vw_id": "[VW ID]"
}
```

#### Response

```
{
  "data": {
    "correlationId": "...",
    "command": "stop_climate",
    "result": 0
  }
}
```
> To START the climate system simply change the JSON value of `active` from boolen false to boolean true. To keep it active, but just change the target temperature, submit with `active` set to true and change the `target_temperature` to an int value representing fahrenheit degrees. (the api happens to expect fahrenheit degrees from me, but that expectation may be localized to me. you might need to submit celsius degrees. experiment with this.)

## Enabling/Disabling Unplugged Climate Control
If you have an EV and you like to live dangerously, you can let your car prepare the climate for you or run the defroster _even when it's not plugged in_. To change this setting and allow it to use battery power to do so, here's how:

#### Request
```
PUT https://b-h-s.spr.us00.p.con-veh.net/mps/v1/vehicles/[ACCOUNT NUMBER]/settings/unplugged_climate_control HTTP/2
content-type: application/json;charset=UTF-8
x-user-id: [USER ID]
x-user-agent: mobile-ios
x-app-uuid: [RANDOM UUID]
authorization: Bearer [ACCESS TOKEN]

{
  "enabled": true,
  "email": "[EMAIL ADDRESS]",
  "tsp_token": "[TSP TOKEN]",
  "vw_id": "[VW ID]"
}
```

### Response
```
{
  "data": {
    "correlationId": "...",
    "command": "enable_unplugged_climate_control",
    "result": 0
  }
}
```
> Issue the same request but with `enabled` set to boolean false in order to turn off unplugged climate control. 

## Retreiving Vehicle Health Report Data 
Get the "health report" data for a vehicle.

#### Request
```
GET https://b-h-s.spr.us00.p.con-veh.net/vhs/v2/vehicle/[VEHICLE ID] HTTP/2
authorization: Bearer [ACCESS TOKEN]
```
### Response
```
{
  "data": {
    "odoReadCount": 55000,
    "odoUOM": "km",
    "vehicleId": "abcdef12-3456-7890-abcd-abcdef0123456",
    "vhrTimeStamp": 1597203545000,
    "overallPriorityCode": "D",
    "overAllPriorityDescription": "ALL_GOOD",
    "applicableCategories": [
      {
        "categoryCode": "DRVR_ASSIST_SYS",
        "categoryDesc": "Driver Assistance Systems",
        "categoryPriorityCode": "D",
        "diagnosticMessages": []
      },
      {
        "categoryCode": "ELECTRIC_SYS",
        "categoryDesc": "Electric System",
        "categoryPriorityCode": "D",
        "diagnosticMessages": []
      },
      {
        "categoryCode": "ENG_TRANS_PWTR",
        "categoryDesc": "Engine & Transmission & Powertrain",
        "categoryPriorityCode": "D",
        "diagnosticMessages": []
      },
      {
        "categoryCode": "TIRES_BRAKES",
        "categoryDesc": "Tires & Brakes",
        "categoryPriorityCode": "D",
        "diagnosticMessages": []
      }
    ],
    "vhrMaintEvents": [
      {
        "distance": 21000,
        "daysCount": 72,
        "eventStatusCode": "2",
        "eventStatusDesc": "Service in",
        "eventType": "INSPECTION",
        "eventTypeDesc": "Service inspection",
        "uom": "km",
        "uomDesc": "Kilometer"
      }
      ...
    ],
    "priorityCodeLegend": [
      {
        "priorityCode": "A",
        "priorityDescription": "Red - Stop! Do not drive vehicle, get professional assistance"
      },
      {
        "priorityCode": "B",
        "priorityDescription": "Yellow - Please see your authorized Volkswagen dealer or an authorized Volkswagen Service Facility for needed repair or adjustment"
      },
      {
        "priorityCode": "C",
        "priorityDescription": "Blue - Please see your authorized Volkswagen dealer or an authorized Volkswagen Service Facility for regular service and maintenance"
      },
      {
        "priorityCode": "D",
        "priorityDescription": "Green - Vehicle OK"
      },
      {
        "priorityCode": "E",
        "priorityDescription": "White"
      }
    ]
  }
}
```
## Requesting a Refresh of the Vehicle Health Report
If you feel like the health report data has gotten stale, you can explicitly ask to refresh it like this:

#### Request
```
POST https://b-h-s.spr.us00.p.con-veh.net/mps/v1/vehicles/[ACCOUNT NUMBER]/health/fresh HTTP/2
content-type: application/json;charset=UTF-8
x-user-id: [USER ID]
x-user-agent: mobile-ios
x-app-uuid: [RANDOM UUID]
authorization: Bearer [ACCESS TOKEN]

{
  "email": "[EMAIL ADDRESS]",
  "tsp_token": "[TSP TOKEN]",
  "vw_id": "[VW ID]"
}
```
#### Response
```
{
  "data": {
    "requested": true
  }
}
```
***
## This Is Almost Everything
There are a few more mappable features of the Car-Net App that I didn't include here, such as setting and enabling Boundary Alerts, Curfew Alerts, Speed Alerts and Valet Alerts. If you really want me to figure those out, let me know and I will. 

### But "it doesn't work for me!" or "my responses look different!", etc.
The responses you receive from your own requests to the Car-Net API may look a little different than the examples given here depending on your vehicle, its features, and the status of your Car-Net account. The account I am testing with is my own paid account, and the car attached to it is a 2016 eGolf. Furthermore, these instructions might possibly only work for VW Car-Net users *in North America*, or it may even be limited to *just customers in the United States*. I don't actually know for sure. I can only test these details out with my own personal account and my personal vehicle, because those are the only ones I have access to. So if these details seem overly ev-centric, that's why. If you'd like me to add features that you see in your Car-Net app but don't see here, send me a message and we'll talk about how we could work that out. 

### Contact Me
I can't offer any real support for the API itself obviously because it belongs to VW and I have no actual control over it, but if there's an issue in my documentation or if something isn't behaving the way I describe it to, please use the Issues tab above. Contact me [@varwwwhtml](https://twitter.com/varwwwhtml) or tom AT itsmetomsmith DOT com) if you have questions, ideas, or just want to talk about this subject. I'd love to know if this document helped you make something interesting. 
