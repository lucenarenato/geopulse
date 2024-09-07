GeoPulse is a specialized server application designed to handle real-time geolocation data from clients via the UDP protocol. The server captures location coordinates and processes them for various uses, such as real-time tracking and historical data retrieval. GeoPulse is optimized utilizing [geojson](https://geojson.org/), [Swoole](https://github.com/swoole/swoole-src) to handle numerous simultaneous UDP connections with low latency. It integrates seamlessly with [Laravel's](https://laravel.com/) queue system allowing you to capture tracking events using laravel jobs.
# Why 
HTTP Location Updates (Slower)

1.Client Sends HTTP Request: The client sends periodic HTTP post requests to update location.<br/>

2.HTTP Request is Sent to Server: The request is transmitted over HTTP.<br/>

3.Server Receives HTTP Request: The server processes the incoming HTTP request.<br/>

4.Server Processes Request (Overhead): The server processes the request with potential delays.<br/>

5.Server Sends HTTP Response Back to Client: The server sends a response back to the client.<br/>

6.Client Receives HTTP Response (OK).<br/>

<br/>
<b>GeoPulse with UDP (Fast)</b><br/><br/>

1.Client Sends UDP Packet (Continuous): The client continuously sends UDP packets.<br/>

2.GeoPulse Server Processes UDP Packet Immediately: Data is processed immediately.<br/>

3.No Response Needed (Data Delivered Fast): No ACk is required from the server.<br/>
# Protocol
GeoPulse uses a JSON format to transmit data packets over UDP for real-time location tracking. The structure is designed to include all necessary information such as the application ID, client ID, and location data, making it easy to parse and process on the server side.<br/>
Example JSON Structure
```javascript
{
  "appId": "yourAppId123",
  "clientId": "client456",
  "data": {
    "type": "Point",
    "coordinates": [102.0, 0.5]
  }
}
```
<br/>
Data Compression: For bandwidth efficiency, you might consider compressing the JSON payload using MessagePack (https://msgpack.org/) GeoPulse already supports MessagePack as an alternative to JSON for smaller payload sizes.

<br/>

# Installation
```
docker pull laggounewalid/geopulse:1.0
```
```
docker run -d -p 9505:9505/udp -v ./pulse-config:/var/www/html/config laggounewalid/geopulse:1.0
```
pulse-config/ folder must contain pulse.php config file
## Requirements
- Queue server supported by illuminate/queue
- Database supported by illuminate/database (Oracle Database not supported by geopulse)
## Database table
```sql
CREATE TABLE
  `pulse_coordinates` (
    `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
    `created_at` timestamp NOT NULL DEFAULT current_timestamp(),
    `appId` varchar(255) DEFAULT NULL,
    `clientId` varchar(255) DEFAULT NULL,
    `coordinate` point DEFAULT NULL,
    `updated_at` timestamp DEFAULT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4
```
## Configuration
Here is boilerplate php code for configuration you can use in pulse-config/pulse.php folder : 
```php
<?php

return [
    /*
        * Configuration: appId
        *
        * The unique application identifier used to authenticate clients.
        * This App ID allows the server to verify that a client is authorized to send UDP packets to GeoPulse.
        * Every UDP packet sent by a client must include this App ID for validation purposes.
    */
    'appId' => 'your_app_id_here',
    /*
        * Configuration: port
        *
        * The application port number for the UDP server.
        * This port is used to receive incoming packets containing coordinate data.
        * Ensure that the specified port (default: 9505) is open and not blocked by a firewall.
    */
    'port' => 9505,
    /*
        * Configuration: use-msgPack
        *
        * A boolean flag to enable or disable MessagePack for data serialization and deserialization.
        * When set to true, the server will use MessagePack to unpack received data packets.
        * If set to false, the server will process data as raw strings.
    */
    'use-msgPack' => true,

    'swoole' => [
        'debug_mode' => true,
        'display_errors' => true,
        'worker_num' => 4,
        'enable_coroutine' => true,
        'open_eof_check' => true,
        'package_eof' => "\r\n",
        'dispatch_mode' => 1,
    ],
    /*
        * Configuration: enable-queue
        *
        * Determines whether the server should packets to your application using queues.
    */
    'enable-queue' => true,

    /*
        * Configuration: queue-pool-size
        *
        * This parameter specifies the number of queue worker connections that will be created 
        * and managed in the Swoole connection pool. These connections are utilized to push 
        * jobs to the server efficiently, ensuring that a new connection does not need to be established 
        * for each queue operation.
        *
        * A larger pool size can help manage a higher volume of queue jobs concurrently, but 
        * it also increases memory and resource usage.
        *
        * Default value: 10
    */
    'queue-pool-size' => 10,

    /*
        * Configuration: queue-connection
        *
        * Defines the connection settings for the queue driver. Follows the same configuration structure
        * used by Laravel's queue system to ensure compatibility and flexibility.
    */
    'queue-connection' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => 'geopulse',
        'retry_after' => 90,
        'block_for' => null,
        'after_commit' => false,
    ],

    'redis' => [
        'options' => [
            'cluster' => 'redis',
            'prefix' => 'YOUR_APP_NAME'.'_database_',
        ],
        'default' => [
            'url' => '',
            'host' => '',
            'username' => '',
            'password' => '',
            'port' => 6379,
            'database' => '0',
        ],
    ],

    /*
        * Configuration: enable-database
        *
        * Determines whether the server should use a database for storing location records.
        * When set to true, the server will persist each location data to the specified database table.
    */
    'enable-database' => true,

    /*
        * Configuration: db-pool-size
        *
        * This parameter defines the number of database connections that will be created 
        * and added to the Swoole connection pool. These connections are reused to handle 
        * database operations efficiently, reducing the overhead of establishing new connections 
        * for each operation.
        *
        * Increasing the pool size can improve performance when dealing with many concurrent 
        * database operations, but it will also increase resource consumption.
        *
        * Default value: 10
    */
    'db-pool-size' => 10,

    /*
        * Configuration: database-connection
        *
        * Defines the database connection settings, following the same structure as Laravel's
        * database configuration file. This ensures compatibility with Laravel's database handling
        * capabilities and allows for flexible configuration across different environments.
    */
    'table-name' => 'pulse_coordinates',
    'database-connection' => [
        'driver' => 'mariadb',
        'url' => null,
        'host' => 'YOUR_DATABASE_HOT',
        'port' => '3306',
        'database' => 'DB_NAME',
        'username' => 'DB_USER',
        'password' => 'DB_PASSWORD',
        'unix_socket' => null,
        'charset' => 'utf8mb4',
        'collation' => 'utf8mb4_unicode_ci',
        'prefix' => '',
        'prefix_indexes' => true,
        'strict' => false,
        'engine' => null,
    ],

];

```
# Example client in php (msgpack enabled)
```php
<?php

use Swoole\Coroutine\Client;

use function Swoole\Coroutine\run;

run(function () {
    $client = new Client(SWOOLE_SOCK_UDP);
    if (! $client->connect('0.0.0.0', 9505, 0.5)) {
        echo "connect failed. Error: {$client->errCode}\n";
    }
    $data = ['appId' => 'your_app_id_here', 'clientId' => '22f8e456-93f2-4173-8f2d-8a010abcceb1', 'data' => ['type' => 'Point', 'coordinates' => [1, 1]]];
    $data = msgpack_pack($data);
    $client->send($data);
    $client->close();
});

```

# Laravel job
```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class PulseLocationUpdatedJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * Create a new job instance.
     */
    public function __construct()
    {
        //
    }

    /**
     * Execute the job.
     */
    public function handle($job, array $data): void
    {
        // data : [
        //     "point" => array:2 [
        //       "type" => "Point"
        //       "coordinates" => array:2 [
        //         0 => 1
        //         1 => 1
        //       ]
        //     ]
        //     "appId" => "your_app_id_here"
        //     "clientId" => "22f8e456-93f2-4173-8f2d-8a010abcceb1"
        //   ]
        $job->delete();
    }
}

```

<br/>

# Cloning and implementing your own broadcasters
To implement your own broadcaster example (kafaka or mongodb ...) simply you need to add in bin/bootstrap.php your broadcaster needs to be located in src/app/Actions/ and must implemnet PacketActionContract
```php
<?php

namespace Pulse\Actions;

use Illuminate\Queue\Capsule\Manager as Queue;
use Pulse\Contracts\Action\PacketActionContract;
use Pulse\Contracts\PacketParser\Packet;

class PublishToKafkaTopic implements PacketActionContract
{
    public function handle(Packet $packet): void
    {
        // 
    }
}

```
```php
$container->add(BroadcastPacketService::class, function () use ($config) {
    $broadcaster = new BroadcastPacketService;
    $broadcaster->addAction(new PublishToKafkaTopic);
    return $broadcaster;
});
```
