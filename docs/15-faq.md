# Stock API Frequently Asked Questions

<!-- MarkdownTOC -->

- [General](#general)
  - [What image sizes are available?](#what-image-sizes-are-available)
  - [How do I download a comp image?](#how-do-i-download-a-comp-image)
- [Licensing](#licensing)
  - [How do I add license references?](#how-do-i-add-license-references)
- [Print on Demand](#print-on-demand)
  - [How do you license assets more than once?](#how-do-you-license-assets-more-than-once)
  - [Why do I see Premium and Video in my search results if I don't have credits?](#why-do-i-see-premium-and-video-in-my-search-results-if-i-dont-have-credits)
  - [How do I filter out Premium content?](#how-do-i-filter-out-premium-content)

<!-- /MarkdownTOC -->

## General

### What image sizes are available?

<ul>
<li><code>110</code>: Small (110 px)
<li><code>160</code>: Medium (160 px)
<li><code>240</code>: Large (240 px)
<li><code>500</code>: Extra large (XL) (500 px). Returned with watermark. (default)
<li><code>1000</code>: Extra-extra large (XXL) (1000 px). Returned with watermark.</li>
</ul>

See [Search API reference](api/11-search-reference.md).

### How do I download a comp image?
There are two kinds of preview images available: cached thumbnail images from the CDN, and non-cached comp images which need to be downloaded from the API. The first type of images are most common, and recommended for most applications. This is a sample URL:
[https://t4.ftcdn.net/jpg/00/84/66/63/240_F_84666330_LoeYCZ5LCobNwWePKbykqEfdQOZ6fipq.jpg](https://t4.ftcdn.net/jpg/00/84/66/63/240_F_84666330_LoeYCZ5LCobNwWePKbykqEfdQOZ6fipq.jpg)

For best performance, use this type of image when possible. In some circumstances, however, you may need the "comp" image version instead. This image requires a different workflow. First you must get the URL from the API, and then download it, often with an authentication header.

- Get comp URL from media ID
```
  GET /Rest/Media/1/Search/Files?search_parameters[media_id]=143738171&result_columns[]=comp_url HTTP/1.1
  Host: stock.adobe.io
  X-Product: MySampleApp/1.0
  x-api-key: MyApiKey
  Authorization: Bearer MyAccessToken
```

- Result
```json
"files": [
    {
        "comp_url": "https://stock.adobe.com/Rest/Libraries/Watermarked/Download/143738171/5"
    }
```

- Curl download request for comp image
```shell
curl "https://stock.adobe.com/Rest/Libraries/Watermarked/Download/143738171/5" \
  -H "x-api-key: YourApiKeyHere" \
  -H "x-product: MySampleApp/1.0" \
  -H "authorization: Bearer AccessTokenHere"
```

## Licensing

### How do I add license references?
License references are extra metadata that you can add to a license record at the time you license the asset. They can be used to track the customer, purchase order, project code, etc., and can be made mandatory or optional. If mandatory, you _must_ include those fields when licensing the asset or you will receive an error.

There are two steps involved:
1. Get a list of required and optional fields from a Member/Profile request
2. Send a POST request to Content/License, including the fields as JSON

In step 1, you call Member/Profile as described in [Licensing assets and stuff](getting-started/apps/06-licensing-assets.md)

```http
  GET /Rest/Libraries/1/Member/Profile?content_id=172563501&license=Standard HTTP/1.1
  Host: stock.adobe.io
  X-Product: MySampleApp/1.0
  x-api-key: MyApiKey
  Authorization: Bearer MyAccessToken
```

If license references have been enabled, the response will include a `cce_agency` array. In the example below, `id:2` ("project reference") is required, while `id:4` ("client reference") is not. Therefore, you must at a minimum include a value for `id:2`.

```json
    "cce_agency": [
        {
            "id": 2,
            "text": "Enter project reference...",
            "required": true
        },
        {
            "id": 4,
            "text": "Enter client reference...",
            "required": false
        }
    ]
```

Instead of calling `GET` Content/License, your application will `POST` Content/License, and set the content type to `application/json`. The body of the message will include your license reference array as shown below.

```http
  POST /Rest/Libraries/1/Content/License?content_id=172563501 HTTP/1.1
  Host: stock.adobe.io
  Content-Type: application/json
  X-Product: MySampleApp
  x-api-key: <API KEY>
  Authorization: Bearer <TOKEN>

  {
    "cce_agency": [
      { "id": "2", "value": "Project Banana" },
      { "id": "4", "value": "King Kong Co" }
    ]
  }
```

To learn how to add or edit license reference fields, see [Edit a product profile for Adobe Stock](https://helpx.adobe.com/enterprise/using/adobe-stock-enterprise.html#CreateeditaproductprofileforAdobeStock).

## Print on Demand

### How do you license assets more than once?
Or, "if my contract requires me to license the same asset again, will this happen automatically in the API?"
No, currently you must set your application to use the `license_again=true` flag. Best practice in this case is to use this flag _every time_ you license an asset, even if it has not been licensed before. The Stock API will only deduct one license from your pool of credits, even if this is the first time you are licensing this asset.

```shell
curl "https://stock.adobe.io/Rest/Libraries/1/Content/License?content_id=112670342&license=Standard&license_again=true" \
  -H "x-api-key: YourApiKeyHere" \
  -H "x-product: MySampleApp/1.0" \
  -H "authorization: Bearer AccessTokenHere"
```

If using the Stock SDK for PHP, add `license_again` to the request object.

```php
    $license_request = new LicenseRequest();
    $license_request->setLicenseState('STANDARD');
    $license_request->setContentId(112670342);
    $license_request->license_again = true;
    $adobe_stock_client = new AdobeStock($api_key, $app_name, 'PROD', $http_client);
    $license_response = $adobe_stock_client->getContentLicense($license_request, $access_token);
```

### Why do I see Premium and Video in my search results if I don't have credits?
Premium assets are included in search results by default. In fact, the default API search includes _all asset types_ (except Editorial--see below). Further, to increase performance, search requests are designed to be anonymous, not requiring authentication. Therefore it is not possible for the Search API to know all your contract details and entitlements when you perform a search--this would slow down the search. Therefore, if you don't have rights to these assets and don't want to see them in search results, you will need to filter them out. See next question, below.

### How do I filter out Premium content?
The Search API includes all asset types by default, therefore you must (1) tell the API not to include Premium assets, and (2) tell it what kind of assets you _do_ want to include. Keep in mind that Video, Templates, and 3D are not considered "Premium," and therefore would still appear in the results.

The following request will exclude Premium assets (`[premium]=false`), and limit results to photos, vectors and illustrations.

```http
GET /Rest/Media/1/Search/Files?locale=en_US
&search_parameters[filters][premium]=false&search_parameters[filters][content_type:photo]=1&search_parameters[filters][content_type:illustration]=1&search_parameters[filters][content_type:vector]=1 HTTP/1.1
Host: stock.adobe.io
X-Product: MySampleApp/1.0
X-API-Key: YourApiKeyHere
```

See [Search API reference](api/11-search-reference.md).
