.. _settings-guide:

Settings Guide
##############

OpenSphere's base settings live in ``config/settings.json``. Unfortunately, JSON does not allow comments.

Basics
======
The entire settings tree falls under two categories:

.. code-block:: none

  {
    "admin": {
      ...
    },
    "user": {
      ...
    }
  }


admin
-----
Anything from ``admin`` is loaded "fresh" (but still follows browser caching directives) every time the application starts. The vast majority of all config goes here.

user
----
The ``user`` section is for specifying defaults for values that the user can otherwise permanently change in their settings. Developers can specify defaults in code:

.. code-block:: javascript

  // the second argument is essentially the application default
  var bgColor = /** @type {string} */ (os.settings.get('bgColor', 'black'));

That value may never actually show up in ``config/settings.json``, but sysadmins can change the default by placing a value like so:

.. code-block:: json

  {
    "user": {
      "bgColor": "white"
    }
  }

Assuming the user has a GUI to change that value, they can still change it to whatever they like, but the default will now be ``"white"``.

Merging
=======

When OpenSphere is built, ``settings.json`` files from all over are merged into one final file which is placed in ``opensphere/dist/opensphere/config/settings.json``. Take the following workspace example:

.. code-block:: none

  workspace/
    opensphere/
    opensphere-config-deployment-base/
    opensphere-config-deployment-specific/
    opensphere-plugin-x/
    opensphere-plugin-y/

By default, the merge order of the configs is defined in the order in which they are resolved. For the example above, this naturally results in the following merge order:

.. code-block:: javascript

  [
    // our base build project
    "opensphere",

    // plugins are resolved next
    "opensphere-plugin-x",
    "opensphere-plugin-y",

    // followed by config last
    "opensphere-config-deployment-base",
    "opensphere-config-deployment-specific"
  ]

You can see the merge order in ``.build/settings-debug.json``, which is what the debug build output uses to load the files in the proper order.

If this order is not satisfactory, each project can define its own merge priority in ``package.json:build.priority``.

Merge Values
============
The merge in the build is performed entirely by the `config plugin`_ of the resolver project.

.. _config plugin: https://github.com/ngageoint/opensphere-build-resolver/blob/master/plugins/config/index.js

Only objects accept merges. Everything else is a replacement:

.. code-block:: javascript

  var original = {
    "name": "Katie",
    "age": 29,
    "interests": ["dogs", "skiing"],
    "likesColors": {
      "blue": true,
      "orange": false
    }
  };

  var newInfo = {
    "name": "Katie Smith",
    "age": 30,
    "interests": ["netflix"],
    "height": 150,
    "likesColors": {
      "purple": true,
      "orange": true
    }
  };

  // merge newInfo to original results in
  var merged = {
    "name": "Katie Smith",
    "age": 30,
    "interests": ["netflix"],
    "height": 150,
    "likesColors": {
      "blue": true,
      "orange": true,
      "purple": true
    }
  };

To delete a value, simply assign the value ``"__delete__"``:

.. code-block:: javascript

  var moreInfo = {
    "likesColors": "__delete__"
  }

  // merge moreInfo to our previously merged object results in
  var merged2 = {
    "name": "Katie Smith",
    "age": 30,
    "interests": ["netflix"],
    "height": 150
  };

Settings
========

Here we will go through some of the most important settings individually. If you have any questions on a more minor one, let us know and we will try to get to it soon.

proxy
-----
``admin.proxy``

The proxy is a failover for getting around mixed content and CORS warnings/errors from other servers.

* ``url`` The URL to the proxy; must contain ``{url}`` e.g. ``https://cors-anywhere.herokuapp.com/{url}``
* ``methods`` The list of http methods supported by the proxy. e.g. ``["GET", "POST", ...]``
* ``schemes`` The schemes supported by the proxy. e.g. ``["http", "https"]``
* ``encode`` whether or not to URL-encode the entire URL when replacing ``{url}``. Defaults to ``true``.

Some items can be configured to use the proxy by default (such as basemaps and some providers). However, for most requests, they will first be tried as a normal request and only try the proxy after that request fails.

providers
---------
``admin.providers``

The providers section is the meat of any OpenSphere configuration. This provides all of the data available to the user by default. While they can certainly add their own data servers, a well-curated list is much more likely to keep users coming back.

Usage:

.. code-block:: javascript

  {
    "admin": {
      "providers": {
        "unique-id-1": {
          "type": "geoserver", // or any provider type
          // ... rest of provider-specific config
        },
        // ... more providers
      }
    }
  }

The design there should be simple and clear. However, let's do a specific example. The ``config/settings.json`` file in OpenSphere itself does a good job of showing how to set up the ``basemap`` provider. So we will add a couple of others:

.. code-block:: json

  {
    "admin": {
      "providers": {
        "arc-sample-server": {
          "type": "arc",
          "label": "ArcGIS Online",
          "url": "//services.arcgisonline.com/ArcGIS/rest/services/"
        },
        "demo-geoserver": {
          "type": "geoserver",
          "label": "Demo Geoserver",
          "url": "https://demo.geo-solutions.it/geoserver/ows"
        }
      }
    }
  }

plugins
-------
``admin.plugins``

This is an object map of plugin IDs to booleans that allows you to disable a plugin entirely in config rather than having to build a new version of the application without that plugin.

Say we wanted to disable KML for some reason:

.. code-block:: json

  {
    "admin": {
      "plugins": {
        "kml": false
      }
    }
  }

baseProjection
--------------
``user.baseProjection``

The ``baseProjection`` sets the default projection of the application. This projection should have a corresponding set of default map layers configured in the ``basemap`` provider. OpenSphere ships with support for EPSG:4326 and EPSG:3857 out of the box. Other projections can be added via config.

Users can change this value in Settings > Map > Projection or by adding a tile layer that is in a projection other than the current projection (assuming that ``enableReprojection`` is ``false``).

.. code-block:: json

  {
    "user": {
      "baseProjection": "EPSG:4326"
    }
  }

metrics
-------
``admin.metrics``

OpenSphere has a metrics API that can be used to gather stats about usage. We want to stress that these metrics are *not sent anywhere*. If you would like to have your metrics sent somewhere, you will have to write a plugin to upload them and add that to your OpenSphere build.

However, for the overly paranoid:

.. code-block:: json

  {
    "admin": {
      "metrics": {
        "enabled": false
      }
    }
  }

terrain
-------
``admin.providers.basemap.maps.terrain``

OpenSphere supports a number of terrain formats out of the box, and more can be added via plugins. By default, the following are supported:

* STK Terrain Server
* WMS Tiles (``image/bil`` only)
* `Cesium World Terrain`_ (requires a `Cesium Ion`_ access token)

Enabling terrain in OpenSphere requires adding a basemap provider to settings with the requisite configuration for the terrain provider type. All providers should set ``"type": "Terrain"`` to identify it as a terrain server, and set ``baseType`` to one of ``cesium``, ``wms``, or ``cesium-ion``. Examples of configuring each are as follows.

STK Terrain Server
******************

.. code-block:: json

  {
    "admin": {
      "providers": {
        "basemap": {
          "maps": {
            "terrain": {
              "type": "Terrain",
              "baseType": "cesium",
              "options": {
                "url": "<server url>"
              },
            }
          }
        }
      }
    }
  }

WMS Tiles
******************

.. code-block:: json

  {
    "admin": {
      "providers": {
        "basemap": {
          "maps": {
            "terrain": {
              "type": "Terrain",
              "baseType": "wms",
              "options": {
                "url": "<server url>"
                "layers": [{
                  "layerName": "<elevation layer>",
                  "minLevel": 0,
                  "maxLevel": 10
                }]
              }
            }
          }
        }
      }
    }
  }

Where the `<server url>` would be something like `https://yourgeoserver:8080/geoserver/ows`
and the `layers` entries have corresponding layer names and resolution ranges.

Cesium World Terrain
********************

A `Cesium Ion`_ account is required to use Cesium World Terrain. After signing in to an Ion account:

1. Select a Cesium World Terrain asset from My Assets.
2. Set the ``options.assetId`` value to the asset ID (displayed in the code example).
3. Navigate to Access Tokens.
4. Set the ``options.accessToken`` value to a token with read access for the terrain asset.

Example:

.. code-block:: javascript

  {
    "admin": {
      "providers": {
        "basemap": {
          "maps": {
            "terrain": {
              "type": "Terrain",
              "baseType": "cesium-ion",
              "options": {
                "accessToken": "<ion access token>",
                "assetId": 1
              }
            }
          }
        }
      }
    }
  }

.. _Cesium World Terrain: https://cesium.com/content/cesium-world-terrain/
.. _Cesium Ion: https://cesium.com/ion/

columnActions
-------------
``admin.columnActions``

Column Actions are specific actions that are performed when displaying field data from a feature.

The only action Opensphere currently supports is the ``UrlColumnAction``.  When displaying feature information, this action will scan column names and values via regular expression to create a clickable link.

The value of the matched field will be substituted for ``%s`` in the ``"action"`` string.

Example:

.. code-block:: json

  {
    "admin": {
      "columnActions": {
        "googleMapsAction": {
          "type": "url",
          "description": "Create a link to a location on Google Maps.",
          "regex": {
            "col": "^DD_LAT_LON$",
            "val": "[\-\d]\d+\.\d+",
            "search": "\w",
            "replace": ","
          },
          "action": "https://www.google.com/maps/@%s"
        }
      }
    }
  }

One of ``"col"`` or ``"val"`` is required. ``"search"`` and ``"replace"`` can be used to transform the text before substituting into the URL, but are entirely optional.

While this is only executed on visible cells in Slickgrid, it is applied to every field's column and value and may cause performance problems depending on the complexity and specificity of the regular expression. If you know specific columns that you want to apply actions to, and know their format in advance, it may be better to simply match on the exact column name.
