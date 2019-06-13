# DX commons Webpack

This module provide shared dependencies using [dll](https://webpack.js.org/plugins/dll-plugin/) for webpacked js application. These components will be used by webpack.

## Core dependencies

To avoid dependency duplication into jahia, this project reference all dependencies shared by all Jahia projects.

Each dependency in `dependencies` section of the `package.json` _MUST_ be loaded in all apps.

## Set up

### Maven

add a maven dependency to this project:

```
<dependency>
    <groupId>org.jahia.modules</groupId>
    <artifactId>dx-commons-webpack</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <type>json</type>
    <classifier>manifest</classifier>
</dependency>
```

And use copy plugin:

```
<plugin>
   <groupId>org.apache.maven.plugins</groupId>
   <artifactId>maven-dependency-plugin</artifactId>
   <executions>
       <execution>
           <id>copy</id>
           <phase>initialize</phase>
           <goals>
               <goal>copy</goal>
           </goals>
       </execution>
   </executions>
   <configuration>
       <artifactItems>
           <artifactItem>
               <groupId>org.jahia.modules</groupId>
               <artifactId>dx-commons-webpack</artifactId>
               <type>json</type>
               <classifier>manifest</classifier>
           </artifactItem>
       </artifactItems>
   </configuration>
</plugin>
```

This provides the manifest that contains all shared packages while building the application or the extension.

### Set webpack configuration to use DLL and CopyWebpackPlugin to provide package.json

The output exposed of the extension must be a single file named `*.bundle.js`

```
output: {
            path: path.resolve(__dirname, 'src/main/resources/javascript/apps/'),
            filename: 'content-editor-ext.bundle.js',
            chunkFilename: '[name].content-editor.[chunkhash:6].js'
        }
```

Plugin configuration:

```
  plugins: [
    new webpack.DllReferencePlugin({
      manifest: require('./target/dependency/dx-commons-webpack-1.0.0-SNAPSHOT-manifest')
    }),
    new CopyWebpackPlugin([{ from: './package.json', to: '' }])
  ]
```

### Define the modules that can be extended

Set apps that are extended in the `package.json` file using the following format

```
  "dx-extends": {
    "@jahia/content-manager": "^1.2.0"
  },
```

### Add the loader to the main application

Use the tag `<js:loader/>` to load the shared resources, the loader script and expose available extensions
The parameter tag `target` defines the window variable that will receive the array of extensions (default is `target`)
Use that array to call the loader script (`bootstap()`) with the application as last entry of the array, as follow:

```
    <js:loader target="cmm-extends"/>
...
    window['cmm-extends'].push('/modules/content-media-manager/javascript/apps/cmm.bundle.js');
    bootstrap(window['cmm-extends']);
```

In the current state, the extensions found will provide the first file named `javascript/apps/*.bundle.js` found in the extension DX bundle

## Execution Sequence

![sequence](docs/img/webpack-common-execution-sequence.svg)
[source][webpack-common-execution-sequence]

**explanations:**
The call to `bootstrap()` will first load all the shared js modules provided by `DX Common Webpack`

In `/dx-commons-webpack/src/main/resources/javascript/js-load.js`:

```
var scriptTag = document.createElement('script');
    scriptTag.src = window.contextJsParameters.contextPath + '/modules/dx-commons-webpack/javascript/apps/commons.bundle.js';
```

Then load all the extensions

In `/dx-commons-webpack/src/javascript/main.js`:

```
    let jsloads = js.slice(0, js.length - 1).map(path => {
        return jsload(path);
    });
```

To finally loads the application itself:

```
    Promise.all(jsloads).then(() => {
        jsload(js.slice(js.length - 1));
    });
```

Once everything is loaded, the application works as if all resources were provided by one single module.

The module `dx-common-webpack` will provide a manifest `dx-commons-webpack-x.x.x-SNAPSHOT-manifest.json` that will be used by application and extensions to know the shared packages.
The packages provided by the module are those defined in `/dx-commons-webpack/src/javascript/main.js` as `require` statement:

```
require('@jahia/apollo-dx');
require('@jahia/design-system-kit');
require('@jahia/i18next');
require('@jahia/icons');
require('@jahia/react-apollo');
require('@jahia/react-material');
require('@jahia/registry');
```

[webpack-common-execution-sequence]: https://mermaidjs.github.io/mermaid-live-editor/#/edit/eyJjb2RlIjoic2VxdWVuY2VEaWFncmFtXG5QYXJ0aWNpcGFudCBicm93c2VyIGFzIEJyb3dzZXJcblBhcnRpY2lwYW50IHJlYWN0IGFzIEpTIEFwcFxuUGFydGljaXBhbnQgZXh0ZW5zaW9ucyBhcyBFeHRlbnNpb25zXG5QYXJ0aWNpcGFudCBsb2FkZXIgYXMgRFggQ29tbW9uXG4gXG5icm93c2VyLT4-cmVhY3Q6IHN0YXJ0XG5yZWFjdC0-PmxvYWRlcjogYm9vdHN0cmFwKG1vZHVsZXMgdG8gbG9hZClcbmxvYWRlci0-PmxvYWRlcjogTG9hZCBzaGFyZWQgcGFja2FnZXNcbmxvYWRlci0tPj5icm93c2VyOiBzaGFyZWQgcGFja2FnZXMgbG9hZGVkXG5sb2FkZXItPj5leHRlbnNpb25zOiBsb2FkIGV4dGVuc2lvbnNcbmV4dGVuc2lvbnMtLT4-YnJvd3NlcjogRXh0ZW5zaW9ucyBsb2FkZWRcbmxvYWRlci0-PnJlYWN0OiBsb2FkIHBhY2thZ2VzIGFwcGxpY2F0aW9uXG5yZWFjdC0tPj5icm93c2VyOiBhcHBsaWNhdGlvbiBMb2FkZWRcbk5vdGUgb3ZlciBicm93c2VyOiBSdW4gQXBwbGljYXRpb25cbiAgICAiLCJtZXJtYWlkIjp7InRoZW1lIjoiZGVmYXVsdCJ9fQ

## Loading screen

This module also contains a loading screen that can be setup to make users wait for large scripts or pages to initialize.
It works by using the GWT ModuleGWTResources extension mechanism to load small scripts that will display immediately a
loading screen until all the content of the GWT iframe are loaded.

Here's an example of how to activate the loading screen from a module (it is deactivated by default) :

In `src/main/resources/META-INF/spring/my-module.xml`:

```
   <bean class="org.jahia.ajax.gwt.helper.ModuleGWTResources">
        <property name="javascriptResources">
            <list>
                <value>/modules/my-module/javascript/activate-loader.js</value>
            </list>
        </property>
    </bean>
```

In `src/main/resources/javascript/activate-loader.js`:

```
    (function () {

        // the following is just an example but you should have a condition here otherwise it will apply to all the GWT engines
        if (window.location.href.indexOf('/cms/my-module-mode') === -1) {
            return;
        }

        function waitForFunction() {
            if (window.displayDXLoadingScreen) {
                window.displayDXLoadingScreen({
                    "en" : "Loading My Module...",
                    "fr" : "Chargement de My Module...",
                    "de" : "My Module wird geladen...",
                });
            } else {
                console.log("Waiting for 100ms for function window.displayDXLoadingScreen to become available");
                setTimeout(waitForFunction, 100);
            }
        }

        waitForFunction();

    })();

```
