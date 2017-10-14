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
Once you've installed the plugin and the the web application, start the web application using node/npm and ensure it is running at http://localhost:3000 (or whever you are hosting it).

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

### Creating Template Variables
Just for the sake of example we'll discuss how to setup a basic test using the demo data in data.json. Create a new dashboard and [configure its templating](http://docs.grafana.org/reference/templating/).


