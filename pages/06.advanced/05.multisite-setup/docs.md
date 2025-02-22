---
title: Multisite Setup
taxonomy:
    category: docs
---

> A multisite setup allows you to create and manage a network of multiple websites, all running on a single installation.

Grav has built-in multisite support. Unlike the [automatic environment configuration](../environment-config), which lets you define custom environments to support different configurations and scenarios, a multisite setup gives you the power to change the way how and from where Grav loads all its files.

### Requirements for a Grav Multisite Setup

The most important thing you will need to run a Grav multisite network is good website hosting. If you are not planning to create many sites and do not expect many visitors, then you can get away with shared hosting. However due to the nature of multisites, you’d probably need a VPS or dedicated server as your sites grow.

### Setup and installation

Before you begin, you’ll want to be sure your web server is capable of running multiple websites i.e., you have access to your Grav root directory. This is essential since serving multiple websites from the same installation is based on a `setup.php` file located in your Grav root.

#### Quickstart (for Beginners)

Once created, the `setup.php` is called every time a user requests a page. In order to serve multiple websites from one single installation, this script (roughly speaking) has to tell Grav where the files (for the configurations, themes, plugins, pages etc.) for a specific subsites are located.

The provided snippets below setup your Grav installation in such a way that a request like

```
http://<subsite>.example.com   -->   user/sites/<subsite>.example.com
```
or
```
http://example.com/<subsite>   -->   user/sites/<subsite>
```

will use the `user/sites` directory as the base "user" path instead of the `user` directory.

If you choose sub-directories or path based URLs for subsites, then the only thing you need is to create a directory for each subsite in the `user/sites` directory containing at least the required folders `config`, `pages`, `plugins` and `themes`.

If you choose sub-domains for structuring your website network, then you will have to configure (wildcard) sub-domains on your server in addition to the setup of your subsites in your `user/sites` directory.

Either way, decide which setup suits your best.

##### Snippets

For subsites accessible via sub-domains copy the [setup_subdomain.php](setup_subdomain.php) file otherwise for subsites accessible via sub-directories the [setup_subdirectory.php](setup_subdirectory.php) file into your `setup.php`.

**setup_subdomain.php**:
```php
<?php
/**
 * Multisite setup for subsites accessible via sub-domains.
 *
 * DO NOT EDIT UNLESS YOU KNOW WHAT YOU ARE DOING!
 */

use Grav\Common\Utils;

// Get subsite name from sub-domain
$environment = isset($_SERVER['HTTP_HOST'])
    ? $_SERVER['HTTP_HOST']
    : (isset($_SERVER['SERVER_NAME']) ? $_SERVER['SERVER_NAME'] : 'localhost');
// Remove port from HTTP_HOST generated $environment
$environment = strtolower(Utils::substrToString($environment, ':'));
$folder = "sites/{$environment}";

if ($environment === 'localhost' || !is_dir(ROOT_DIR . "user/{$folder}")) {
    return [];
}

return [
    'environment' => $environment,
    'streams' => [
        'schemes' => [
            'user' => [
               'type' => 'ReadOnlyStream',
               'prefixes' => [
                   '' => ["user/{$folder}"],
               ]
            ]
        ]
    ]
];
```

**setup_subdirectory.php**:
```php
<?php
/**
 * Multisite setup for sub-directories or path based
 * URLs for subsites.
 *
 * DO NOT EDIT UNLESS YOU KNOW WHAT YOU ARE DOING!
 */

use Grav\Common\Filesystem\Folder;

// Get relative path from Grav root.
$path = isset($_SERVER['PATH_INFO'])
   ? $_SERVER['PATH_INFO']
   : Folder::getRelativePath($_SERVER['REQUEST_URI'], ROOT_DIR);

// Extract name of subsite from path
$name = Folder::shift($path);
$folder = "sites/{$name}";
$prefix = "/{$name}";

if (!$name || !is_dir(ROOT_DIR . "user/{$folder}")) {
    return [];
}

// Prefix all pages with the name of the subsite
$container['pages']->base($prefix);

return [
    'environment' => $name,
    'streams' => [
        'schemes' => [
            'user' => [
               'type' => 'ReadOnlyStream',
               'prefixes' => [
                   '' => ["user/{$folder}"],
               ]
            ]
        ]
    ]
];
```

#### Advanced configuration (for Experts)

Once created a `setup.php` have access to two important variables: (i) `$container`, which is the yet not properly initialized [Grav instance](https://github.com/getgrav/grav/blob/develop/system/src/Grav/Common/Grav.php) and (ii) `$self`, which is an instance of the [ConfigServiceProvider class](https://github.com/getgrav/grav/blob/develop/system/src/Grav/Common/Service/ConfigServiceProvider.php).

Inside this script you can do anything, but please be aware that the `setup.php` is called every time a user requests a page. This means that memory critical or time-consuming initializations operations lead to a slow-down of your whole system and should therefore be avoided.

In the end the `setup.php` has to return an associative array with the optional environment name **environment** and a stream collection **streams**
(for more informations and in order to set them up correctly, see the section [Streams](#streams)):

```php
return [
  'environment' => '<name>',            // A name for the environment
  'streams' => [
    'schemes' => [
      '<stream_name>' => [              // The name of the stream
        'type' => 'ReadOnlyStream',     // Stream object e.g. 'ReadOnlyStream' or 'Stream'
        'prefixes' => [
          '<prefix>' => [
            '<path1>',
            '<path2>',
            '<etc>'
          ]
        ],
        'paths' => [                    // Paths (optional)
          '<paths1>',
          '<paths2>',
          '<etc>'
        ]
      ]
    ]
  ]
]

```

>>>> Please be aware that a this very early stage you neither have access to the configuration nor to the URI instance and thus any call to a non-initialized class might end in a freeze of the system, in unexpected errors or in (complete) data loss.

#### Streams

In Grav streams are objects, mapping a set of physical directories of the system to a logical device. They are classified via their `type` attribute. For readonly streams that's the `ReadOnlyStream` type and for read-writeable streams that's the `Stream` type. You can register any custom stream type and pointing to it as long as it is an instance of the [StreamInterface](https://github.com/rockettheme/blob/develop/toolbox/StreamWrapper/src/StreamInterface.php) interface class.

Mapping physical directories to a logical device can be done in two ways, either by setting up `paths` or `prefixes`. The former can be understood as a 1-to-1 mapping, whereas the latter (as the name suggests) allows you to combine several physical paths into one logical stream. Let's say you want to register a stream with the name "image". You can then with the stream `images://` list with

```php
'image' => [
    'type' => 'ReadOnlyStream',
    'paths' => [
        'user/images',
        'system/images'
    ]
];
```

all images located in the folders `user/images` and `system/images`. For **prefixes** consider the example

```php
'cache' => [
    'type' => 'Stream',
    'prefixes' => [
        '' => ['cache'],
        'images' => ['images']
    ]
];
```

In this case `cache://` resolves to `cache`, but `cache://images` resolves to `images`.

Last but not least, streams can be used in other streams. For example provided a stream `user` and a stream `system` exists, the above "image" stream can also be written as

```php
'image' => [
    'type' => 'ReadOnlyStream',
    'paths' => [
        'user://images',
        'system://images'
    ]
];

```
