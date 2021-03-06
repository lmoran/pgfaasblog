
# Introducing pgFaaS


pgFaaS stands for *PostGresql Functions As A Service*, and aims at providing a simple way to perform geo-processing on hosted geographic data using a ReSTful API.

At the current stage. this is more of a proof-of-concept than a polished product, but it is stanle enough to be used in production, 
albeit with rough edges when it comes to debugging (error messages are still difficult to understand). 


## What is it for?

To deploy and use Node.js scripts that run SQL queries against PostgreSQL databases over HTTP in a simple and scalable way.

A ReSTful API (and a web user interface) can be used to deploy, update, remove, and invoke scripts, making it easy to develop APIs to power front-end applications.

For the time being, only Node.js can be used to develop scripts, but pgFaaS can be extended to other languages without much effort. 


## Why should I use it?

In short, there is no easier way to to add geo-processing functionality to web applications on hosted data.
And let's not forget how powerful and performing are SQL queries augmented by PostGIS store functions (and there are more than a hundred of such functiona, working on vector, raster, and routing data).    


## How does it work?

A ReSTful API is used to deploy scripts than can access PostgreSQL data and perform geo-processing using PostGIS (SQL statements are embedded into JavScript programs). 
No connection parameters are set at this stage, as eveything is done behind the scenes when configuring pgFaaS. 
Once the script is deployed, it can be invoked using HTTP POST requests with a JSON body (JSON is expected to be returned as well).

For instance, deploying a script contained in a file named `math.json` can be accomplished by running this HTTP POST request:
```bash
curl -XPOST "http://<server>/api/function/namespaces/lmoran"\
  --header "Content-Type:application/json"\
  --data  @math.json
```

As you may have noticed, the JavaScript program has to be encapsulated in a JSON before being deployed, which can be accomplished by storing to a file the ouput of this program:
```javascript
const math = {
  name: 'math',
  sourcecode: require('fs').readFileSync('./functions/script-math.js', 'utf-8'),
  test: {verb: 'add', a: 1, b: 2}
};
console.log(JSON.stringify(math));
``` 

The JSON used for deployment has three properties:
* `name` The function name
* `sourcecode` The script as string
* `test` An example of a body that can be used to test the function (it is used by the  pgFaaS user interface to make testing easier).


Functions are grouped into namespaces (`lmoran`, in this case); namespaces can be created with another POST request like this one:
```bash
curl -XPOST "http://<server>>/api/function/namespaces"\
  --header "Content-Type:application/json"\
  --data  '{"name":"lmoran"}'
```

After the deployment, functions can be called with POST requests.


## How does a script look like?

This is a script to compute simple arithmetic.
```javascript
module.exports = {
  add: (sqlexec, req, callback) => {
    return callback(null, {c: req.body.a + req.body.b});
  },
  minus: (sqlexec, req, callback) => {
    return callback(null, {c: req.body.a - req.body.b});
  },
  times: (sqlexec, req, callback) => {
    return callback(null, {c: req.body.a * req.body.b});
  },
  div: (sqlexec, req, callback) => {
    return callback(null, {c: req.body.a / req.body.b});
  },
  mod: (sqlexec, req, callback) => {
    return callback(null, {c: req.body.a % req.body.b});
  }
};
```

Since there are different functions that can be invoked independently, this is, in effect, an API in a script.

Invoking it to, say, adding two numbers, is just a POST call away:
```bash
curl -XPOST "http://sandbox.pgfaas.aurin.org.au/api/function/namespaces/sample/math"\
  --header "Content-Type:application/json"\
  --data '{"verb": "add", "a": 3 , "b": 6}'
``` 
(Please note the use of the `verb` parameter, that acts as a "switch" to invoke different functions within the script.)


A more meaningful example using SQL:
```javascript
module.exports = {
  knnbusstops: (sqlexec, req, callback) => {
    sqlexec.query(
      `SELECT ST_AsGeoJSON(ST_Transform(way, 4326)) AS geom, name, osm_id AS id, operator,
         ST_Distance(ST_Transform(way, 4326), ST_SetSRID(ST_MakePoint($2, $3), 4326)) AS distance
         FROM planet_osm_point
         WHERE highway = 'bus_stop'
         ORDER BY distance LIMIT $1`,
      [req.body.k, req.body.x, req.body.y], (err, result) => {
        if (err) {
          return callback(err, null);
        }
        let geoJson = {
          type: "FeatureCollection", crs: {
            type: "name",
            properties: {
              name: "urn:ogc:def:crs:EPSG::3857"
            }
          }, features: []
        };
        result.rows.forEach((row) => {
          if (row.geom.type !== "GeometryCollection") {
            geoJson.features.push({
              type: "Feature",
              geometry: JSON.parse(row.geom),
              properties: {
                id: row.id,
                name: row.name,
                operator: row.operator
              }
            });
          }
        });
        return callback(err, geoJson);
      }
    );
  }
};  
```

This function computes the K-nearest bus stops (Open Street Map data) to a point passed in the POST body as `x` and `y` parameters, and returns them as a GeoJSON document.
The SQL statement is passed as a string, with an array of parameters to improve readibility.

More than one statement can be sent, nesting calls to `sq;exec.query` in callbacks. 


## What's under the hood?

Every time a script is deployed, a Docker container is created (or re-created) and then OpenFaaS takes care of scaling it up or down. according to workload.

More specifically, the base Docker image is the same for every pgFaaS container, but the scripts (and dtabase connection parameters) are sent as environment variables to it, making every container run a differnt script without the need to build a different image.   

![pgFaaS deployment diagram](https://raw.githubusercontent.com/lmoran/pgfaasblog/master/architecture.png "pgFaaS deploymemnt diagram")


## SQL access to database a... unsafe, isn't it?

By default, users cannot modify the database and are limited to one schema of one database; in addition Docker containers created by pgFaaS are constrained in what resources (CPUs, RAM) they can consume.

In a further development, scripts could be screened for malicious code during deployment (this feature has not been developed yet.)  


## Where can I get more info?

You can play with a live pgFaaS istance in [the sandbox](http://sandbox.pgfaas.aurin.org.au/ui).
 
More background can be inferred by viewing the presentation I gave at the [FOSS4G SotM Oceania '18 prez](https://www.youtube.com/watch?v=mhZcpuliMxI),
or just read [the slides](https://docs.google.com/presentation/d/1D6HrRwEBD93NiIH4OxK7XgkwGulOHRIpYsuu6mNSG84/edit?usp=sharing) if you're in a hurry.


## Who is behind it?

The [AURIN project](https://aurin.org.au) is a government-backed organization that provides data and analytical tools to urban researchers.


## Kudos

Powered by:
[![PostgreSQL](https://raw.githubusercontent.com/AURIN/pgFaas/master/assets/postgresql.png)](https://www.postgresql.org)
[![OpenFaaS](https://raw.githubusercontent.com/AURIN/pgFaas/master/assets/openfaas.png)](https://www.openfaas.com)

