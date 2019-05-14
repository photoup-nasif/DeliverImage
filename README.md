PhotoUp DeliverImage API V1.0.0

* [Introduction to Restful API endpoints](#introduction-to-restful-api-endpoints)
* [Introduction to DeliverImage API](#introduction-to-deliverimage-api)
* [Connecting Third Party and PhotoUp Accounts](#connecting-third-party-and-photoup-accounts)
* [Authentication](#authentication)
* [Request and Response](#request-and-response)
* [DeliverImage Endpoints: Resources](#deliverimage-endpoints-resources)
* [Example Responses](#example-responses)
* [Parameter Objects](#parameter-objects)
* [Third Party Endpoint Requirements](#third-party-endpoint-requirements)
* [Third Party Duplicate Image Protocols](#third-party-duplicate-image-protocol)
* [Short Flow Summary](#short-flow-summary)

#### Introduction to Restful API endpoints
Endpoints can be access with http verbs: GET, POST, PUT, and DELETE

- GET /home  (get the list of home)
- POST /home (add new home)
- PUT /home/1/rating (update the rating of home with id = 1)
- DELETE /home/1 (delete home with id = 1)

#### Introduction to DeliverImage API
This API is a guide and a requirement for the third party service  provider so that PhotoUp can upload images directly to the third party platform. For Delivery image purposes this API version utilizes or supports only GET POST and PUT methods.

#### Connecting Third Party and PhotoUp Accounts
Clients will login into PhotoUp account and they can link the two accounts once inside PhotoUp. The steps will be as follows
1. The client will click a connection link inside PhotoUp and it will open a new tab showing the thirdparty signup/login page. If the client signs up a new account: example thirdparty signup link ```https://thirdparty.com/signup?pk=somecharacter&sk=somecharacter```
    1. PhotoUp will add a get variables pk and sk in the link. This is the one time Public key and Private key for each Client.
    2. PhotoUp will only save the pk and sk once the link of two accounts is successful.
    	1. When the client choose to signup and when the signup is successful, the client will be informed that the accounts are link. For this to happen the third party will have to call PUT /signup see [DeliverImage Endpoints: Resources](#deliverimage-endpoints-resources) and also add the pk and sk in the payload.
    	2. When the client choose to login, once logged in, the third party also needs to call PUT/signup to inform PhotoUp that the third party got the request and linked the account. 
    3. When the third party has successfully called the PUT /signup, PhotoUp will then close the third party page from no. 1 above and proceed to Delivery form.

#### Authentication
- PhotoUp will generate Public Key and Secret Key for third party client
- This Keys will be used for authenticating every request both for PhotoUp and for the thirdparty
- The API client and PhotoUp must provide PU-API-Public-Key in the header (The Key here is the Public key)
- The API client and PhotoUp must provide PU-API-Timestamp in the header
- The API client and PhotoUp must provide PU-API-Signature in the header
 - The signature is made by hashing components concatenated by a dash
(-) with hmac sha256
   - The signature components are: Public Key and timestamp

```
        protected function connectToPhotoUp($method, $url , $data = array()) {
            $timestamp = time();
            $path = parse_url($url, PHP_URL_PATH);
            $method = strtoupper($method);

            $signature_components = array(PUBLIC_KEY, $timestamp);
            $signature = hash_hmac("sha256", implode("\n", $signature_components), SECRET_KEY);

            $headers = array(
                "PU-API-Public-Key: ".PUBLIC_KEY,
                "PU-API-Timestamp: ".$timestamp,
                "PU-API-Signature: ".$signature,
                "Content-Type: application/json"
            );
            $ch = curl_init();

            curl_setopt($ch, CURLOPT_URL, $url);
            curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
            curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);
            curl_setopt($ch, CURLOPT_HEADER, FALSE);

            if(!empty($data)) {
                curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
            }

            $response = curl_exec($ch);
            $http_status_code = curl_getinfo($ch, CURLINFO_HTTP_CODE);

            curl_close($ch);

            $response = json_decode($response, true);

            return array("response" => $response, "code" => $http_status_code);
        }

        private function isRequestValid() {
            $received_api_key = "";
            $received_api_sign = "";
            $received_api_timestamp = "";

            foreach (getallheaders() as $name => $value) {
                if($name == "PU-API-Public-Key") {
                    $received_api_key = $value;
                } else if ($name == "PU-API-Signature") {
                    $received_api_sign = $value;
                } else if ($name == "PU-API-Timestamp") {
                    $received_api_timestamp = $value;
                }
            }

            $signature_components = array($received_api_key, $received_api_timestamp);
            $expected_signature = hash_hmac("sha256", implode("\n", $signature_components), PUBLIC_KEY);

            if($expected_signature == $received_api_sign) {
                return true;
            }
            return false;
        }
```

#### Request and Response
- Media type format
 Both request and response should be in json format
- Upon request
response should be either in status code
200, 201 or 204
 - with or without response data. If there is a response data the
format would always be in
-  ``` {"message": "success", .....more data if applicable... }```
- On Failure, status code is either in 4xx or 5xx and in format of
-  ``` {"errors": "String of The Error Message" }```

#### DeliverImage Endpoints: Resources
| Endpoint | Param | Description |
|-|-|-|
| PUT /link/<third_party_short_name> | {"sk": "kl12j3jkl12j3h12jh", "pk":"jk2jjk589sthdv614"}| sk and pk are provided by photoup on redirecting to signup/login page of the third party website. PhotoUp will provide the third party short name as a constant when PhotoUp and the thirdparty decided to do an integration. |
|-|-|-|
| GET /home | | List out deliverable homes/properties. For response example See [Example Responses](#example-responses). |
|-|-|-|
| GET /home/1234 || Grab Edited image's URL for home id 1234. For response example See [Example Responses](#example-responses).|
|-|-|-|
| PUT /home/1234/failedDelivery | | Third party must notify PhotoUp when PhotoUp delivery for edited images is not successful. PhotoUp will then look for the problems in delivery and will send delivery/submit request with all of the images again to the third party. See [Third Party Endpoint Requirements](#third-party-endpoint-requirements). See [Parameter Objects](#parameter-objects). |
|| errors | (String) Required. Text description of error(s). |
|| image_ids | (hash) Optional. Affecting image ids. |
|-|-|-|
| PUT /home/1234/successDelivery | | Third party must notify PhotoUp when PhotoUp delivery for edited images is successful. |
|| property_id | (int) Required. ID of property/album on the third party platform. |
|-|-|-|

#### Example responses
- example response when calling ```GET /home```.
```
{
        "homes": [
                {
                        "home_id": 53831,
                        "home_name": "123 Main Street",
                        "home_image_count": 3
                },
                {
                        "home_id": 53831,
                        "home_name": "124 Wealthy Street",
                        "home_image_count": 20
                }
        ]
}
```
- example response when calling ```GET /home/1234```.
```
{
        "home_id": 53831,
        "home_name": "123 Main Street",
        "home_images": [
                {
                        "image_id": 677568,
                        "image_name": "Dining Room.jpg",
                        "image_version": 1,
                        "image_url": "https://www.photoup.net/app/deliverImage/677568"
                },
                {
                        "image_id": 677576,
                        "image_name": "Lawn.jpg",
                        "image_version": 2,
                        "image_url": "https://www.photoup.net/app/deliverImage/677576"
                },
                {
                        "image_id": 677625,
                        "image_name": "Kitchen.jpg",
                        "image_version": 1,
                        "image_url": "https://www.photoup.net/app/deliverImage/677625"
                }
        ]
}
```

#### Parameter Objects
- Post data example for /home/1/failedDelivery
```
{"errors": "Cannot download Image(s)", "image_ids":[12345,1234]}
```


#### Third Party Endpoint Requirements
The endpoints below are requirement endpoint for the third party. PhotoUp will send the notifications/data to the third party when necessary.

Third party will share the same PUBLIC and SECRET keys, such that the third party can implement the same authentication checking. see [Authentication](#authentication)

| Short name | Endpoint | Data from PhotoUp | Description |
|-|-|-|-|
| list property | GET /property | none |  PhotoUp will call this endpoint so that we can show the client the list of property he/she can transfer image to. |
| new property delivery | PUT /property | see below| PhotoUp will call this endpoint when delivering images to the specified tour. This will also create a new property on the third party |
| existing property delivery | PUT /property/12345 | see below| PhotoUp will call this endpoint when delivering images to the specified tour. |
|-|-|-|-|

- Response format for GET /property
```
{
        "properties": [
                {
                        "property_id": 1234,
                        "property_name": "Property Name 1",
                        "property_images_count": 22,
                        "property_url": "https://www.thirdparty.com/property/1234"
                },
                {
                        "property_id": 1289,
                        "property_name": "Property Name 2",
                        "property_images_count": 13,
                        "property_url": "https://www.thirdparty.com/property/1289"
                },
                {
                        "property_id": 1318,
                        "property_name": "Property Name 3",
                        "property_images_count": 35,
                        "property_url": "https://www.thirdparty.com/property/1318"
                },
                {
                        "property_id": 1330,
                        "property_name": "Property Name 4",
                        "property_images_count": 16,
                        "property_url": "https://www.thirdparty.com/property/1330"
                },
        ]
}
```

- Payload for ```PUT /property```
```
{
		"property_id": 1235, // the id of the third party property
        "home_id": 53831,
        "home_name": "123 Main Street",
        "home_images": [
                {
                        "image_id": 677568,
                        "image_name": "Dining Room.jpg",
                        "image_version": 1,
                        "image_url": "https://www.photoup.net/app/deliverImage/677568"
                },
                {
                        "image_id": 677576,
                        "image_name": "Lawn.jpg",
                        "image_version": 2,
                        "image_url": "https://www.photoup.net/app/deliverImage/677576"
                },
                {
                        "image_id": 677625,
                        "image_name": "Kitchen.jpg",
                        "image_version": 1,
                        "image_url": "https://www.photoup.net/app/deliverImage/677625"
                }
        ]
}
```

- Payload for ```PUT /property/12345```
```
{
		"property_name": "My new Third Party property name", // the id of the third party property
        "home_id": 53831,
        "home_name": "123 Main Street",
        "home_images": [
                {
                        "image_id": 677568,
                        "image_name": "Dining Room.jpg",
                        "image_version": 1,
                        "image_url": "https://www.photoup.net/app/deliverImage/677568"
                },
                {
                        "image_id": 677576,
                        "image_name": "Lawn.jpg",
                        "image_version": 2,
                        "image_url": "https://www.photoup.net/app/deliverImage/677576"
                },
                {
                        "image_id": 677625,
                        "image_name": "Kitchen.jpg",
                        "image_version": 1,
                        "image_url": "https://www.photoup.net/app/deliverImage/677625"
                }
        ]
}
```

Third party responses for PUT /property/ OR PUT /property/12345 can be:
```
{"message": "success"}//200 OK
```
```
{"errors": "Posted Format Error"}// status 4xx
```

#### Third Party Duplicate Image Protocol
Photoup provided image_id and version(See [Example Responses](#example-responses)), The third party can determine if succeding transfers have images in the payload that are already on the property.
When duplicate happens, The third party should replace(or delete and add) the image with the sent payload if its a newer version. 
When its not duplicate, the third party should add the image as new image in the property.

#### Short Flow Summary 
1. The Client will need to link the account see [Connecting Third Party and PhotoUp Accounts](#connecting-third-party-and-photoup-accounts).
2. Once the account is connected the client can transfer images from PhotoUp to the third party by:
	1. Pulling images from the third party - The client is on the third party website and then he is shown which PhotoUp 'home' photos to pull and store in the Third party 'property'. The third party needs to inform PhotoUp if the transfer is a success or a failure.
	2. Pushing images from PhotoUp. The client can go to PhotoUp's Home info page and choose which property from a given list to do the transfer. The third party needs to inform PhotoUp if the transfer is a success or a failure.
	

