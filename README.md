
# Introducing pgFaaS


pgFaaS, simply put, is *PostGresql Functions As A Service*

At the current stage is more of a proof-of-concept than a polished product, but it is stanle enought to be used in production, 
albeit with rough edges when it comes to debugging (error messages are still difficult to understand). 


## What is it for?

To deploy and use Node.js scripts that access PostgreSQL databases over HTTP simply, and scalably.

A ReSTful API (and a web user interface) can be used to deploy/uneploy/update and invoke scripts, mkaing it easy to develop API that can power front-end applications.

For the timbeing, only Node.js can be used to develop scripts, but pgFaaS can be extended to toerh languages without much effort. 


## Why should I use it?

If you, like me, want developers to add geo-processing functionality to their applications on the data you host, there is no easier way.  


## How does it work?


## How a script looks like?

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
(Spolier alert, it returns 9.)


A more meaningful example:
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

This computes the K-nearest bus stops in New Caledonia to a point (passed in the POST body as 'x' and 'y' parameters.) 



## Where can I get more info?

You can play with a live pgFaaS istance in [the sandbox](http://sandbox.pgfaas.aurin.org.au).
 
More background can be inferred by viewing the presentation I gave at the [FOSS4G SotM Oceania '18 prez]https://www.youtube.com/watch?v=mhZcpuliMxI
or just read [the slides](https://docs.google.com/presentation/d/1D6HrRwEBD93NiIH4OxK7XgkwGulOHRIpYsuu6mNSG84/edit?usp=sharing) if you're in a hurry.


## Who is behind it?

The [AURIN project](https://aurin.org.au) is a government-backed organization that provides data and analytical tools to urban researchers.


## Kudos

Powered by:

![logo](https://raw.githubusercontent.com/AURIN/pgFaas/master/assets/postgresql.png "PostgreSQL")
![logo](https://raw.githubusercontent.com/AURIN/pgFaas/master/assets/openfaas.png "OpenFaaS")

