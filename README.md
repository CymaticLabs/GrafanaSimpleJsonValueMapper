# Grafana SimpleJson Value Mapper
A simple Node.js/Express application that provides basic template variable aliasing/value mapping.

## What Is It For?
[Grafana](https://grafana.com/) is a popular data visualization platform that can query [many popular data sources](https://grafana.com/plugins?type=datasource) and visualize their data sets (most often time series data).

#### Template Variables
Once configured, [template variables](http://docs.grafana.org/reference/templating/) allow users to dynamically configure dashboards without having to be technically savvy enough to write or rewrite the underlying queries that power them. This is all driven through an intuitive visual interface within the dashboard itself.

![grafana dashboard](http://docs.grafana.org/img/docs/v4/templated_dash.png)

#### The Problem of Cryptic Values
Many times the template variables that appear in these drop down lists are themselves the result of a dynamic query. Sometimes the values are human-readable, but othertimes they can be quite cryptic and it's hard for the dashboard user to know exactly what it is they are selecting or looking at.

For example, imagine that you are querying a list of IDs that happen to be GUIDs/UUIDs. A single ID might look something like this:

`123e4567-e89b-12d3-a456-426655440000`

That doesn't tell the dashboard user anything about who the ID belongs to. Even if the user could be taught to memorize and match IDs to the things they belong to, as the list of those IDs grows, keeping track of them becomes even more cumbersome and potentially impossible.

#### Aliasing Values to Something More Readable
One solution would be to replace the hard-to-read values with something that is much more easy to read and understand for the user. Unfortunately there are two problems to this approach in Grafana:

1. Grafana has some aliasing options, but they generally do not work for dynamic template values, such as ones that are the results of a query.
2. Even if you could replace the unreadable value with something better, the underlying queries usually need the unreadable value, so the human-readable replacement would not work as a substitute when being fed into the query after the user selects it from the dashboard template menu.

#### Creating a Dynamic Look-Up Service
What is truly needed in these scenarios is a dynamic look-up service that can be given the unreadable values and return readable values for the template UI while also retaining the unreadable values to drive queries dynamically. It turns out that this is actually possible with Grafana, but requires the use of a datasource plugin called [SimpleJson datasource](https://grafana.com/plugins/grafana-simple-json-datasource).

By using the SimpleJson datasource plugin, it is possible to create the dynamic look-up service that will satisfy the needs of human-readable values in the template variable menus while also retaining the unreadable values that generally drive the queries behind the scenes. The SimpleJson datasource turns a web application into a generic datasource in Grafana (which it communicates with via HTTP).

#### Human-Readable Template Variables Solved
This web application implements functionality to solve this look-up problem by implementing the contracts needed by the SimpleJson datasource plugin. It can be used as-is if your aliasing needs are simple and editing a JSON file on disk will suffice. You can also use this project as starting point for reference or by extending it yourself to tap into your own data sources for aliasing on the backend.

## Installation
The following steps will help you setup a basic template variable look-up service using this web application.

**Note**: This installation assumes you have Grafana installed and have sufficient privileges and know how to perform basic administrative tasks.

### Install the SimpleJson Datasource Plugin
Follow the [instructions here](https://grafana.com/plugins/grafana-simple-json-datasource/installation) to install the SimpleJson datasource plugin. You will have to restart Grafana before the datasource plugin will be accessible.

### Install Grafana SimpleJson Value Mapper
Install this project by cloning it or downloading it. By default when run from localhost, it should run on port 3003.

### Configure a SimpleJson datasource
Once you've installed the plugin and the the web application, start the web application using node/npm and ensure it is running at http://localhost:3003 (or whever you are hosting it).

If you've restarted Grafana, login as a user with sufficient privileges and create a new SimpleJson datasource that will talk to the web application.

![setup datasource](https://github.com/CymaticLabs/GrafanaSimpleJsonValueMapper/blob/master/public/images/setup-simplejson-datasource.png?raw=true)


### Configure the Aliasing Data
In the web application you will find the aliasing data file here: `/server/data.json`. You can start out using the test data that already exists in the file just to verify things work or you can edit it to add your own data.

Here's a quick explanation of the test data and basic schema of the data that the web application expects:

```
{
  "list": [
    "value1",
    "value2",
    "value3"
  ],
  "lookup": {
    "value1": "Value 1",
    "value2": "Value 2",
    "value3": "Value 3"
  }
}
```

Each top-level property (*"list"*, *"lookup"*, etc.) of the JSON object act as a separate dataset that you can reference when defining your template variables. This way if you have more than one set of look-up/aliasing data you don't have to put it all into one big unified list.

The first entry, *"list"*, is an array that will only provide a non-aliased set of results back to Grafana's template variable query. Mostly it will be used in this example to emulate dynamic values that were received from a query from another datasource in Grafana. It does have limited usefulness in that it can be filtered with a basic string match using a "contains" approach. More on that later.

Most aliasing datasets will look like the second entry, *"lookup"*, which is a hash/map, where the propery name (*value1*, *value2*, etc.) is the input/unreadable ID and its value is the human-readable alias. You will add as many of these look-up style entries as you need into the data.json file.

Let's assume you had a couple regions of IoT devices for example that have GUIDs as their identifier:

```
{
  "IoT Outside": {
    "03e79c66-d624-459d-9a6c-caa40c428c29": "Camera North",
    "b61a9967-7363-4d6b-a51c-27b5a539eba5": "Camera South",
    "a0caf0b9-e5f0-4a54-b6fe-7eb8a692b3c9": "Camera East",
    "4d84b539-e81c-4341-a97d-aa5f133b6ee3": "Camera West"
  },
  "IoT Inside": {
    "bd683289-ae38-4c09-9826-17c6243b7621": "Temperature Sensor",
    "9513dff8-6b32-4450-88ca-33786a147142": "Living Room Lights",
  }
}
```

Your template variable queries to the configured SimpleJson datasource will reference *"IoT Outside"* or *"IoT Inside"* when doing aliasing look-ups. You can of course always just create a single, unified list; that is entirely up to you. You just need to create one top-level look-up entry/hash.

Make sure to restart the web application after making any edits so that they will be live.

## Creating Template Variables
Just for the sake of example we'll discuss how to setup a basic test using the demo data in data.json. Create a new dashboard and [configure its templating](http://docs.grafana.org/reference/templating/).

First lets setup a template variable that will pull the list values into the first drop down menu:

![setup-list](https://github.com/CymaticLabs/GrafanaSimpleJsonValueMapper/blob/master/public/images/setup-list-variable.png?raw=true)

The *Variable Type* should be: `Query`

The *Data source* property will be the SimpleJson datasource you setup earlier, in this example the `Template Aliasing` datasource.

The query will be: `{ "data": "list" }`, where the *data* property references the top-level property/dataset in the data.json file (in this case: *list*).

It's up to you if you want to include multiple values and the "All" option, but it's recommended to see how more advanced aliasing scenarios might work.

Next setup the look-up template variable:

![setup-lookup](https://github.com/CymaticLabs/GrafanaSimpleJsonValueMapper/blob/master/public/images/setup-lookup-variable.png?raw=true)

The setup here is fairly similar except the query looks like this:

`{ "data": "lookup", "id": "$list" }`.

In this case, since *lookup* is a hash/map in data.json we can include the *id* property which says which ID it should return an alias for. This can be a hardcoded ID or set of IDs, but in this case we are using the *list* template variable directly/dynamically by using: `$list`. This way one template variable will dynamically drive the other.

If you want to hardcode IDs it will look like this:

`{ "data": "lookup", "id": "value1" }` (for single ID)

`{ "data": "lookup", "id": "(value1|value2|value3)" }` (for multiple IDs)

You template variables on your dashboard should look like this:

![setup template variables](https://github.com/CymaticLabs/GrafanaSimpleJsonValueMapper/blob/master/public/images/setup-template-variables.png?raw=true)

### Testing Things Out

Now go back to your dashboard and check out the drop down list for both template variables:

![test variables 1](https://github.com/CymaticLabs/GrafanaSimpleJsonValueMapper/blob/master/public/images/setup-list-test-1.png?raw=true)

![test variables 2](https://github.com/CymaticLabs/GrafanaSimpleJsonValueMapper/blob/master/public/images/setup-list-test-2.png?raw=true)

Selecting values from the first list should filter values from the second list (which in this example is the unreadable IDs; even though, yes, they are still readable):

![filter values](https://github.com/CymaticLabs/GrafanaSimpleJsonValueMapper/blob/master/public/images/setup-list-test-3.png?raw=true)

Now, if you were to use the template variables *$lookup* in a dashboard query, even though the menu labels are *Value 1* and *Value 3*, the actual value that would be injected into the query via the *$lookup* template variable would be *value1* and *value3*. Substitude *value1* and *value3* with a GUID instead and you'll start to get the idea.

### Real World
A more likely scenario is that you will have a hidden template variable (you can opt to hide template variables when creating them) that will query a datasource which returns the unreadable IDs, like the IoT GUIDs in the exampel further up. Then you will pass those IDs over to the aliasing datasource which will have their human-readable names in a hash look-up in data.json. The results of that template variable will be what the user sees, and then you will use that template variable in your dashboard queries to query and visualize data related to those underlying GUIDs, but the user will only ever see the human-readable version of the IDs.

## Security
If you want to secure things a bit more, you can enable HTTP BASIC authentication. When you configure your datasource in Grafana, click the checkbox to enable this and enter your desired user name and password.

When you launch the Node.js web application set the environmental variables `HTTP_AUTH_USERNAME` and `HTTP_AUTH_PASSWORD` to the same values. This way, whenever a request is made to the aliasing service (*/search* in terms of the route), the request will be authenticated against these credentials.

If you need more security you are encouraged to extend this project with your own solution and/or use more sophisticated proxying via Grafana.

## Conclusion
This web application primarily serves as an example only. You might be able to get by with it if your needs are simple and manually updating the data.json file and restarting the web application are not going to be deal-breakers when updates to the look-up data occur. Otherwise you are encouraged to check out the code in /routes/index.js and alter it to fit your needs.

Happy Mapping!
