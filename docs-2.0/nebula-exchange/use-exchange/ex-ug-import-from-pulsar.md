# Import data from Pulsar

This topic provides an example of how to use Exchange to import Nebula Graph data stored in Pulsar.

## Environment

This example is done on MacOS. Here is the environment configuration information:

- Hardware specifications:
  - CPU: 1.7 GHz Quad-Core Intel Core i7
  - Memory: 16 GB

- Spark: 2.4.7, stand-alone

- Nebula Graph: {{nebula.release}}. [Deploy Nebula Graph with Docker Compose](../../4.deployment-and-installation/2.compile-and-install-nebula-graph/3.deploy-nebula-graph-with-docker-compose.md).

## Prerequisites

Before importing data, you need to confirm the following information:

- Nebula Graph has been [installed](../../4.deployment-and-installation/2.compile-and-install-nebula-graph/2.install-nebula-graph-by-rpm-or-deb.md) and deployed with the following information:

  - IP addresses and ports of Graph and Meta services.

  - The user name and password with write permission to Nebula Graph.

- Exchange has been [compiled](../ex-ug-compile.md), or [download](https://repo1.maven.org/maven2/com/vesoft/nebula-exchange/) the compiled `.jar` file directly.

- Spark has been installed.

- Learn about the Schema created in Nebula Graph, including names and properties of Tags and Edge types, and more.

- The Pulsar service has been installed and started.

## Steps

### Step 1: Create the Schema in Nebula Graph

Analyze the data to create a Schema in Nebula Graph by following these steps:

1. Identify the Schema elements. The Schema elements in the Nebula Graph are shown in the following table.

    | Element  | Name | Property |
    | :--- | :--- | :--- |
    | Tag | `player` | `name string, age int` |
    | Tag | `team` | `name string` |
    | Edge Type | `follow` | `degree int` |
    | Edge Type | `serve` | `start_year int, end_year int` |

2. Create a graph space **basketballplayer** in the Nebula Graph and create a Schema as shown below.

    ```ngql
    ## Create a graph space
    nebula> CREATE SPACE basketballplayer \
            (partition_num = 10, \
            replica_factor = 1, \
            vid_type = FIXED_STRING(30));
    
    ## Use the graph space basketballplayer
    nebula> USE basketballplayer;
    
    ## Create the Tag player
    nebula> CREATE TAG player(name string, age int);
    
    ## Create the Tag team
    nebula> CREATE TAG team(name string);
    
    ## Create the Edge type follow
    nebula> CREATE EDGE follow(degree int);

    ## Create the Edge type serve
    nebula> CREATE EDGE serve(start_year int, end_year int);
    ```

For more information, see [Quick start workflow](../../2.quick-start/1.quick-start-workflow.md).

### Step 2: Modify configuration files

After Exchange is compiled, copy the conf file `target/classes/application.conf` to set Pulsar data source configuration. In this example, the copied file is called `pulsar_application.conf`. For details on each configuration item, see [Parameters in the configuration file](../parameter-reference/ex-ug-parameter.md).

```conf
{
  # Spark configuration
  spark: {
    app: {
      name: Nebula Exchange {{exchange.release}}
    }
    driver: {
      cores: 1
      maxResultSize: 1G
    }
    cores: {
      max: 16
    }
  }


  # Nebula Graph configuration
  nebula: {
    address:{
      # Specify the IP addresses and ports for Graph and all Meta services.
      # If there are multiple addresses, the format is "ip1:port","ip2:port","ip3:port".
      # Addresses are separated by commas.
      graph:["127.0.0.1:9669"]
      meta:["127.0.0.1:9559"]
    }

    # The account entered must have write permission for the Nebula Graph space.
    user: root
    pswd: nebula

    # Fill in the name of the graph space you want to write data to in the Nebula Graph.
    space: basketballplayer
    connection: {
      timeout: 3000
      retry: 3
    }
    execution: {
      retry: 3
    }
    error: {
      max: 32
      output: /tmp/errors
    }
    rate: {
      limit: 1024
      timeout: 1000
    }
  }
  # Processing vertices
  tags: [
    # Set the information about the Tag player.
    {
      # The corresponding Tag name in Nebula Graph.
      name: player
      type: {
        # Specify the data source file format to Pulsar.
        source: pulsar
        # Specify how to import the data into Nebula Graph: Client or SST.
        sink: client
      }
      # The address of the Pulsar server.
      service: "pulsar://127.0.0.1:6650"
      # admin.url of pulsar.
      admin: "http://127.0.0.1:8081"
      # The Pulsar option can be configured from topic, topics or topicsPattern.
      options: {
        topics: "topic1,topic2"
      }

      # Specify the column names in the player table in fields, and their corresponding values are specified as properties in the Nebula Graph.
      # The sequence of fields and nebula.fields must correspond to each other.
      # If multiple column names need to be specified, separate them by commas.
      fields: [age,name]
      nebula.fields: [age,name]

      # Specify a column of data in the table as the source of VIDs in the Nebula Graph.
      vertex:{
          field:playerid
      }


      # The number of data written to Nebula Graph in a single batch.
      batch: 10

      # The number of Spark partitions.
      partition: 10
      # The interval for message reading. Unit: second.
      interval.seconds: 10
    }
    # Set the information about the Tag Team.
    {
      name: team
      type: {
        source: pulsar
        sink: client
      }
      service: "pulsar://127.0.0.1:6650"
      admin: "http://127.0.0.1:8081"
      options: {
        topics: "topic1,topic2"
      }
      fields: [name]
      nebula.fields: [name]
      vertex:{
          field:teamid
      }
      batch: 10
      partition: 10
      interval.seconds: 10
    }

  ]

  # Processing edges
  edges: [
    # Set the information about Edge Type follow
    {
      # The corresponding Edge Type name in Nebula Graph.
      name: follow

      type: {
        # Specify the data source file format to Pulsar.
        source: pulsar

        # Specify how to import the Edge type data into Nebula Graph.
        # Specify how to import the data into Nebula Graph: Client or SST.
        sink: client
      }

      # The address of the Pulsar server.
      service: "pulsar://127.0.0.1:6650"
      # admin.url of pulsar.
      admin: "http://127.0.0.1:8081"
      # The Pulsar option can be configured from topic, topics or topicsPattern.
      options: {
        topics: "topic1,topic2"
      }

      # Specify the column names in the follow table in fields, and their corresponding values are specified as properties in the Nebula Graph.
      # The sequence of fields and nebula.fields must correspond to each other.
      # If multiple column names need to be specified, separate them by commas.
      fields: [degree]
      nebula.fields: [degree]

      # In source, use a column in the follow table as the source of the edge's source vertex.
      # In target, use a column in the follow table as the source of the edge's destination vertex.
      source:{
          field:src_player
      }

      target:{
          field:dst_player
      }

      # (Optional) Specify a column as the source of the rank.
      #ranking: rank

      # The number of data written to Nebula Graph in a single batch.
      batch: 10

      # The number of Spark partitions.
      partition: 10

      # The interval for message reading. Unit: second.
      interval.seconds: 10
    }

    # Set the information about the Edge Type serve
    {
      name: serve
      type: {
        source: Pulsar
        sink: client
      }
      service: "pulsar://127.0.0.1:6650"
      admin: "http://127.0.0.1:8081"
      options: {
        topics: "topic1,topic2"
      }

      fields: [start_year,end_year]
      nebula.fields: [start_year,end_year]
      source:{
          field:playerid
      }

      target:{
          field:teamid
      }

      # (Optional) Specify a column as the source of the rank.
      #ranking: rank

      batch: 10
      partition: 10
      interval.seconds: 10
    }
  ]
}
```

### Step 3: Import data into Nebula Graph

Run the following command to import Pulsar data into Nebula Graph. For a description of the parameters, see [Options for import](../parameter-reference/ex-ug-para-import-command.md).

```bash
${SPARK_HOME}/bin/spark-submit --master "local" --class com.vesoft.nebula.exchange.Exchange <nebula-exchange-{{exchange.release}}.jar_path> -c <pulsar_application.conf_path>
```

!!! note

    JAR packages are available in two ways: [compiled them yourself](../ex-ug-compile.md), or [download](https://repo1.maven.org/maven2/com/vesoft/nebula-exchange/) the compiled `.jar` file directly.

For example:

```bash
${SPARK_HOME}/bin/spark-submit  --master "local" --class com.vesoft.nebula.exchange.Exchange  /root/nebula-exchange/nebula-exchange/target/nebula-exchange-{{exchange.release}}.jar  -c /root/nebula-exchange/nebula-exchange/target/classes/pulsar_application.conf
```

You can search for `batchSuccess.<tag_name/edge_name>` in the command output to check the number of successes. For example, `batchSuccess.follow: 300`.

### Step 4: (optional) Validate data

Users can verify that data has been imported by executing a query in the Nebula Graph client (for example, Nebula Graph Studio). For example:

```ngql
GO FROM "player100" OVER follow;
```

Users can also run the [`SHOW STATS`](../../3.ngql-guide/7.general-query-statements/6.show/14.show-stats.md) command to view statistics.

### Step 5: (optional) Rebuild indexes in Nebula Graph

With the data imported, users can recreate and rebuild indexes in Nebula Graph. For details, see [Index overview](../../3.ngql-guide/14.native-index-statements/README.md).
