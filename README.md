### Google Cloud Function ([getting started](https://medium.com/@timhberry/getting-started-with-python-for-google-cloud-functions-646a8cddbb33))

To create a Google Cloud Function we need a [Google Cloud Project](https://cloud.google.com/resource-manager/docs/creating-managing-projects) and [gcloud SDK](https://cloud.google.com/sdk/docs/).

If we have already some projects we can check them by typing:

`gcloud projects list`
```
PROJECT_ID        NAME                     PROJECT_NUMBER
gef-ld-toolbox    gef-ld-toolbox           1080184168142
gfw-apis          Global Forest Watch API  872868960419
resource-watch    Resource Watch           312603932249
skydipper-196010  skydipper                230510979472
soc-platform      SOC Platform             345072612231
```
and select one by:

`gcloud config set project gef-ld-toolbox`

On the example folder we have set the 2 ways we work with cloud functions. Javascript and Python.

## Python 

You will need to add a main.py file with your function

```python
import ee
import json
with open("privatekey.json") as keyfile:
    extracted_key_data = keyfile.read()
    credentials = ee.ServiceAccountCredentials(key_data=extracted_key_data)
ee.Initialize(credentials, use_cloud_api=True)
#service_account = 'gef-ldmp-server@gef-ld-toolbox.iam.gserviceaccount.com'
#credentials = ee.ServiceAccountCredentials(service_account, 'privatekey.json')
#ee.Initialize(credentials)

def serialize_response(url):

    return {
        'download_url': url
    }

def download_image(request):
    # For more information about CORS and CORS preflight requests, see
    # https://developer.mozilla.org/en-US/docs/Glossary/Preflight_request
    # for more information.
    # Set CORS headers for the main request
    headers = {
        'Access-Control-Allow-Origin': '*',
        'Content-Type': 'application/json'
    }
    # Set CORS headers for the preflight request
    if request.method == 'OPTIONS':
        # Allows GET requests from any origin with the Content-Type
        # header and caches preflight response for an 3600s
        headers.update({
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
            'Access-Control-Allow-Headers': 'Content-Type',
            'Access-Control-Max-Age': '3600'
        })

        return ('', 204, headers)

    

    request = request.get_json()
    if 'geometry' in request:
        polygon = ee.Geometry.Polygon(request['geometry'].get('features')[0].get('geometry').get('coordinates'))
        image = ee.Image(request['assetId']).clip(ee.Geometry(polygon))
    else:
        image = ee.Image(request['assetId'])
    url = image.getDownloadUrl()

    return (json.dumps(serialize_response(url)), 200, headers)
```
In the same directory include the `privatekey.json` with the [service account keys](https://cloud.google.com/iam/docs/creating-managing-service-account-keys) and the `requirements.txt` file. 

Finally cd to that directory and deploy the cloud Function with the following command:

`gcloud beta functions deploy <function Name> --runtime python37 --trigger-http`

## Javascript

You will need to add a index.js file with your function
```javascript
const ee = require('@google/earthengine');
const PRIVATE_KEY = require('./privatekey.json');


const serialize = (originalData) => {
  return JSON.stringify({
    rows: [originalData]
  });
};

const interact_raster = (input) => {
  const reducer = {
    "reducer":ee.Reducer.first(), 
    "geometry":ee.Geometry.Point(input.point, 'EPSG:4326'), 
    "bestEffort":true, 
    "maxPixels":10e8, 
    "tileScale":10
  }
  return ee.Image(input.asset_id).reduceRegion(reducer);
};

exports.rasterInteraction = (req, res) => {
  const body = req.body
  res.set('Access-Control-Allow-Origin', '*');
  res.set('Content-Type', 'application/json');
  

  if (req.method === 'OPTIONS') {
   // Send response to OPTIONS requests
   res.set('Access-Control-Allow-Methods', 'POST, GET, OPTIONS');
   res.set('Access-Control-Allow-Headers', 'Content-Type');
   res.set('Access-Control-Max-Age', '3600');
   res.status(204).send('');
  }

  ee.data.authenticateViaPrivateKey(
    PRIVATE_KEY,
    () => {
      ee.initialize(null, null, () => {
        const result = interact_raster(body);
        result.evaluate((json) => res.status(200).send(serialize(json)));
      });
    }, 
    e => console.error(`Authentication error: ${e}`)
  );
};

```


In the same directory include the `privatekey.json` with the [service account keys](https://cloud.google.com/iam/docs/creating-managing-service-account-keys) and the `package.json` file. 

You can test it adding to your `package.json` file the next line 
```json  
"devDependencies": {
    "@google-cloud/functions-framework": "^1.2.1"
  },
"scripts": {
    "start": "functions-framework --target=rasterInteraction"
  },
```
Doing a  `npm install` and running the server with `npm start` and check `http://localhost:8080/`

`gcloud beta functions deploy <function Name> --runtime nodejs10 --trigger-http`
	
To deploy it cd to that directory and deploy the cloud Function with the following command:



Note that the cloud function name matches the name of the function we defined in code: `<function Name>`.