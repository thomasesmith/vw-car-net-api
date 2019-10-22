> *HEY! As of October 13th, 2019–and after VW changed their API and replaced their Car-Net app with a new one on October 22nd–all of this information is outdated and no longer works. I am going to let this document sit here for reference, and I will try to make a similar one that reflects the changes made to their API as soon as I can figure out how to.*

# Connecting to the Volkswagen Car-Net API

I used [mitmproxy](https://mitmproxy.org) to discover/reverse-engineer some of the endpoints and general behavior of how the Volkswagen Car-Net mobile app consumes its RESTful API. This document is intended to be instructions on how to use some of its functions for your own projects related to your car, untethered from the official Car-Net app, and without relying on html scrapers.

This README will stay code-agnostic and just detail each http request.

## Disclaimer (Kind Of)
The responses you receive from your own requests to the Car-Net API may look a little different than the examples given here, depending on your vehicle, its features, and the status of your Car-Net account. The account I am testing with is my own paid account, and the car attached to it is a 2016 eGolf. Furthermore, these instructions might possibly only work for VW Car-Net users *in North America*, or it may even be limited to *just customers in the United States*. I don't actually know for sure. I can only test these details out with my own personal use-case. So if these details seem overly EV-centric, that's why. 

It's worth mentioning also that **I have no idea if using the API in this manner violates your terms of service with VW Car-Net. So assume the worst, and proceed at your own risk.**

## Contact
I can't offer any real support for this API, because it's not mine and I have no control over it, but contact me ([@varwwwhtml](https://twitter.com/varwwwhtml) or tom AT itsmetomsmith DOT com) if you have any questions, ideas, or thoughts regarding this subject. I'd love to know if this document helped someone start something.

***
## Like Everything, You Start by Logging In
You have to first log in in order to receive a token to use for the rest of your requests. But logging in is actually two steps, just like it is in the app: first to log in to your account with your email address and password, then a second level to authorize with your account number and PIN. 

#### Request
First, send a request like this one:

````
POST https://cns.vw.com/mps/v1/login
Content-Type:application/json

{
    "email": "{YOUR CAR-NET ACCOUNT EMAIL ADDRESS}",
    "password": "{YOUR PASSWORD}"
}
````
> The "Content-Type:application/json" header is required in every `POST`, `PATCH`, and `PUT` request that is made to this server. I've found that if you don't include it, you'll just get back 415 errors. The `GET`s don't require it, however.

#### Response
Upon a successful login, you will receive back a JSON object containing your account details that will look something like this: 
````
{
  "value": {
    "needs_password_reset": false,
    "token": {
      "value": "{YOUR TOKEN, USUALLY 48 BYTES}",
      "expiration_date": "2019-10-31T17:09:09.932Z"
    },
    "account": {
      "vehicles": [
        {
          "id": "{YOUR CAR-NET ACCOUNT NUMBER}",
          "vin": "WVWKP7AU123456789",
          "make": "VW",
          "model": {
            "type": "egolf_2016",
            "name": "e-Golf"
          },
          "year": 2016,
          "color": "Night Blue Metallic",
          "subscription": {
            "state": "ok",
            "message": "Your subscription will end on `10/10/2020.",
            "active_features": [
              "battery",
              "climate",
              "departure_timers",
              "remote",
              "navigation",
              "alerts",
              "health_report"
            ]
          },
          "vehicle_features": [
            "door_lock",
            "battery",
            "climate"
          ],
          "dealership": {
            "name": "Sim City Volkswagen",
            "phone": "8185551212",
            "location": {
              "latitude": 12.1234,
              "longitude": -123.123
            },
            "address": {
              "countryOnly": false,
              "street": "123 Fake St.",
              "city": "Sim City",
              "state": "CA",
              "zip_code": "91001",
              "country": "USA"
            }
          }
        }
      ]
    }
  }
}
````
>TAKE NOTE of the "token" and "id" values (your "id" is also your Car-Net account number), you will need both of these in order to complete all the other API requests detailed in this document.  

...then you must authorize your Car-Net account number and PIN. This secondary account/PIN authorization process must be performed in order for your token to successfully be used in any the other API requests. Trying to make API requests without doing so will result in errors. Here's how:
#### Request 
````
POST https://cns.vw.com/mps/v1/vehicles/{YOUR ACCOUNT NUMBER}/auth
Content-Type:application/json
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{THE TOKEN GIVEN TO YOU BY THE PREVIOUS RESPONSE}

{
    "id": "{YOUR ACCOUNT NUMBER}",
    "pin": "{YOUR PIN}"
}
````

#### Response
The response will be simply the expiration date/time of your token. 
````
{
  "value": {
    "token_expiration_date": "2019-09-16T16:46:55.039Z"
  }
}
````

> This token expiry is set for about 15 minutes in to the future, but I have found that it's not a "firm" expiration time for your token. It seems more to be the expiration time if you were to then stop making requests. Each request you make with your token seems to lengthen your tokens expiry time to 15 more minutes in to the future. But if you wait 15 minutes between your last and next request using your token, your requests will then produce token expiry errors. 

>If any of your requests suddenly begin responding with a 500 error ("Something went wrong"), or a 403 response with "Vehicle session expired" message, simply re-perform the account/PIN authorization request, **using the same token you were first issued**. This will "refresh" the token, and issue it a new expiry time of 15 minutes in the future. You can then continue to use it for requests. In this instance, **you don't have to repeat the first email/password log in request and get a new token.** 

>I notice also that including the `Cookie` header with the value `SERVERID=mps-server-001` seems to be necessary for reliable use. 

***

## View Your Vehicle's "Status"
Now that you're all logged in and authorized, you can begin poking around.  Start by getting your cars current "status." The status is a summary of your vehicle that includes a lot of stuff: your car's GPS coordinates, its odometer reading, its charging status and approximate driving range, etc..

#### Request
````
GET https://cns.vw.com/mps/v1/vehicles/{YOUR ACCOUNT NUMBER}/status
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{YOUR TOKEN}
````

#### Response
The API will respond with a JSON object summarizing your vehicle's status: 
````
{
  "value": {
    "timestamp": "2019-09-16T16:23:29.000Z",
    "current_mileage": 27283,
    "next_maintenance_milestone": {
      "mileage_interval": 0,
      "absolute_mileage": 12600,
      "date": "2019-11-09T17:26:18.623Z"
    },
    "exterior": {
      "entry": {
        "secure": true,
        "doors": {
          "front_left_open": false,
          "front_right_open": false,
          "rear_left_open": false,
          "rear_right_open": false,
          "rear_open": false
        },
        "windows": {
          "front_left_open": false,
          "front_right_open": false,
          "rear_left_open": false,
          "rear_right_open": false,
          "engine_hood_open": false
        }
      },
      "lights": {
        "left_on": false,
        "right_on": false
      }
    },
    "power": {
      "cruise_range": 87,
      "cruise_range_units": "MILES",
      "battery": {
        "charge_percent_remaining": 76,
        "minutes_until_full_charge": 15,
        "charge_plug": "unplugged",
        "triggered_by_timer": false
      }
    },
    "climate": {
      "outdoor_temperature": 73,
      "target_temperature": 64,
      "climate_on": false,
      "defrost_on": false,
      "unplugged_climate_control_enabled": true,
      "triggered_by_timer": false
    },
    "location": {
      "latitude": 123.123456,
      "longitude": -123.123456
    },
    "address": {
      "countryOnly": false,
      "street": "123 CurrentStreetAddress Drive",
      "city": "Sim City",
      "state": "CA",
      "zip_code": "91001",
      "country": "USA"
    }
  }
}
````
## Force Refresh of Vehicle Status
Sometimes the vehicle status that gets returned is a little stale (as evident by an old `timestamp` value), but this tends to be when nothing about the car is changing much. For instance, while charging my eGolf, the /status response updates regularly and doesn't require a forced refresh, but when the car is parked in a way that nothing about the car is really changing, it lets the status response get a little old. 

If you want to force the API to poll the vehicle again and get an updated summary, do the following: 

#### Request
````
GET https://cns.vw.com/mps/v1/vehicles/{ACCOUNT NUMBER}/status/fresh
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{YOUR TOKEN}
````
#### Response
````
{
  "value": {
    "transaction_id": "...",
    "command": "fetch_vehicle_status"
  }
}
````
>Just like many of the functions below, the response you get from this will just be an acknowlegment you've sent a request. In order to get back the actual updated status object, you will have to request your vehicle's status again after you make this request, and it might take a few seconds for that to begin returning the updated summary. 

>NOTE! If you make this request too frequently though, you will begin getting 429 Too Many Request errors from the server when you do it. I haven't yet figured out exactly how often is too often. 


## View Your Vehicle's Settings
In addition to "status," you can also get a summary of the current values of each setting in your vehicle:
#### Request
````
GET https://cns.vw.com/mps/v1/vehicles/{ACCOUNT NUMBER}/settings
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{YOUR TOKEN}
````
It will respond with a JSON object containing your car's setting values:
#### Response
````
{
  "value": {
    "primary_notification_methods": {
      "email": [
        "your-email-address@mail.com"
      ],
      "phone": [
        "8055551213"
      ]
    },
    "boundaries": [],
    "departure_timers": [
      {
        "id": 1,
        "enabled": false,
        "profile": {
          "name": "Name of Timer 1",
          "charging": true,
          "climate_on": false
        },
        "expired": false,
        "recurring_schedule": {
          "days_of_week": [
            "monday",
            "tuesday",
            "wednesday",
            "friday"
          ],
          "time": {
            "hours_minutes": "11:00",
            "time_zone": "America/New_York"
          }
        }
      },
      {
        "id": 2,
        "enabled": false,
        "profile": {
          "name": "Name of Timer 2",
          "charging": true,
          "climate_on": false
        },
        "expired": false,
        "recurring_schedule": {
          "days_of_week": [
            "thursday"
          ],
          "time": {
            "hours_minutes": "11:00",
            "time_zone": "America/New_York"
          }
        }
      },
      {
        "id": 3,
        "enabled": false,
        "profile": {
          "name": "Name of Timer 3",
          "charging": true,
          "climate_on": false
        },
        "expired": false,
        "recurring_schedule": {
          "days_of_week": [
            "monday",
            "tuesday",
            "wednesday",
            "thursday",
            "friday",
            "saturday",
            "sunday"
          ],
          "time": {
            "hours_minutes": "11:00",
            "time_zone": "America/New_York"
          }
        }
      }
    ],
    "max_charge_current": "max",
    "unplugged_climate_control_enabled": true
  }
}
````

***
## Various Functions I've Been Able to Map So Far
Both `PUT` and `PATCH` requests are used to adjust the status and settings of your vehicle. I don't know why they made some PUT and some PATCH, so keep a keen eye on which is used for which request. 

Here are the functions I've mapped out so far...

### Locking/Unlocking The Doors
#### Lock Doors Request
````
PUT https://cns.vw.com/mps/v1/vehicles/{ACCOUNT NUMBER}/status/exterior/doors
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{YOUR TOKEN}
Content-Type:application/json

{
  "lock": true
}
````
#### Lock Doors Response
````
{
  "value": {
    "transaction_id": "...",
    "command": "lock_doors"
  }
}
````
>Just a note about all of these functions: getting a response doesn't mean your request has already been fulfilled in the real world. In fact, if you were to run this request, then immediately following it request the "status" of the car, you will not see in the status response that the car's doors are now locked. It takes me about 30 seconds for that to actually happen. The API server itself responds quickly, but it seems to take some time for the vehicle to actually lock, and then even more time for the "status" summary request to reflect that has been locked. This is the case with all of the `PUT` and `PATCH` requests made to the API. I assume this is a limitation of the speed of the data connection to the vehicle itself, and the speed at which the vehicle can process directives sent to it from VWs servers (it's not like these API requests communicate directly with the vehicle itself). Perhaps this explains somewhat the excruciating slowness of the Car-Net app? 

And then to unlock...
#### Unlock Doors Request
````
PUT https://cns.vw.com/mps/v1/vehicles/{ACCOUNT NUMBER}/status/exterior/doors
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{YOUR TOKEN}
Content-Type:application/json

{
  "lock": false
}
````
#### Unlock Doors Response
````
{
  "value": {
    "transaction_id": "...",
    "command": "unlock_doors"
  }
}
````

### Flashing Headlights/Honking Horn
#### Request
````
PUT https://cns.vw.com/mps/v1/vehicles/{YOUR ACCOUNT NUMBER}/status/exterior/horn_and_lights
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{YOUR TOKEN}
Content-Type:application/json
{
  "horn": true,
  "lights": true
}
````
#### Response
````
{
  "value": {
    "transaction_id": "...",
    "command": "flash_lights_and_honk_horn"
  }
}
````
>You can also JUST honk the horn, by changing the JSON to `"lights": false`, or you can JUST flash the lights by changing the JSON to `"horn": false`.

### Stop Charging
#### Request
````
PATCH https://cns.vw.com/mps/v1/vehicles/{YOUR ACCOUNT NUMBER}/status/charging
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{YOUR TOKEN}
Content-Type:application/json
{
  "active": false
}
````
> Watch out! Unlike the others, charging related requests are of type `PATCH`.

#### Response
````
{
  "value": {
    "transaction_id": "...",
    "command": "stop_charge"
  }
}
````
### Start Charging

#### Request
````
PATCH https://cns.vw.com/mps/v1/vehicles/{YOUR ACCOUNT NUMBER}/status/charging
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{YOUR TOKEN}
Content-Type:application/json
{
  "active": true
}
````
> Watch out! Unlike the others, charging related requests are of type `PATCH`.

#### Response
````
{
  "value": {
    "transaction_id": "...",
    "command": "start_charge"
  }
}
````
### Setting the Maximum Charging Current
#### Request 
````
PATCH https://cns.vw.com/mps/v1/vehicles/{YOUR ACCOUNT NUMBER}/settings/max_charge_current
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{YOUR TOKEN}
Content-Type:application/json
{
  "max_charge_current": "max"
}
````
> The values that the API will accept for "max_charge_current" probably differ depending on your vehicle, but I happen to know that for the 2016 eGolf SE (w/ the optional DC Fast Charging package) the acceptable values of "max_charge_current" are 5, 10, 13, and "max". 
#### Response
````
{
  "value": {
    "transaction_id": "...",
    "command": "set_max_charge_current"
  }
}
````

### Turning On/Off Climate Control and Adjusting Target Temperature
If your vehicle is plugged in, or if you have "unplugged_climate_control_enabled" set to `true`, you can adjust the climate control with the API. Here's how: 
#### Request
````
PUT https://cns.vw.com/mps/v1/vehicles/{YOUR ACCOUNT NUMBER}/status/climate
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{YOUR TOKEN}
Content-Type:application/json
{
  "active": true,
  "target_temperature": 72
}
````
> Issue the same request but with `"active": false` to turn **off** climate control. 

>"target_temperature" is the number in fahrenheit (at least for me) that one would set as the temperature they want the interior of the car to be. I have not tested the limits of what is an acceptable range of values for "target_temperature".
#### Response
````
{
  "value": {
    "transaction_id": "...",
    "command": "start_climate"
  }
}
````
> If you issued that request as `"active": false` the "command" value would've been returned as "stop_climate". 

### Turning On/Off Defroster
If your vehicle is plugged in, or if you have "unplugged_climate_control_enabled" set to `true`, you can remotely turn off or on the defroster. 
#### Request
````
PUT https://cns.vw.com/mps/v1/vehicles/{YOUR ACCOUNT NUMBER}/status/defrost
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{YOUR TOKEN}
Content-Type:application/json
{
  "active": true
}
````
> Issue the same request but with `"active": false` to turn **off** the defroster. 
#### Response
````
{
  "value": {
    "transaction_id": "...",
    "command": "start_defrost"
  }
}
````
> If you issued that request as `"active": false` the "command" value would've been returned as "stop_defrost". 


### Enabling/Disabling Unplugged Climate Control
If you have an EV and like to live dangerously, you can let your car prepare the climate for you or run the defroster even when it's not plugged in. To change this setting to allow it to use battery power to do so, here's that:

#### Request
````
PUT https://cns.vw.com/mps/v1/vehicles/{YOUR ACCOUNT NUMBER}/settings/unplugged_climate_control
Cookie:SERVERID=mps-server-001
X-CarNet-Token:{YOUR TOKEN}
Content-Type:application/json
{
  "enabled": true
}
````
> Issue the same request but with `"enabled": false` to turn **off** the unplugged climate control. 
#### Response
````
{
  "value": {
    "transaction_id": "...",
    "command": "enable_unplugged_climate_control"
  }
}
````
> If you issued that request as `"enabled": false` the "command" value would've been returned as "disable_unplugged_climate_control". 
***
## What I'd Still Like To Figure Out
- **Minimum Battery Level** I don't understand why my car has some settings that can be changed in the Car-Net mobile app itself, and other settings for which the app has no control, and instead it instructs me to log in to the Car-Net browser app to adjust. "Minimum Battery Level" (the daftly named point to which your battery will charge, before stopping itself) is one such setting that the iOS mobile app can't natively control, but the browser app can. It would be very convenient to figure out how I can use this API to change that setting. I couldn't sniff out an endpoint for it during the mitmproxy process because of the mobile apps lack of an adjustment control for it. It's also strange that neither the /status or /settings endpoint responses include this value at all.