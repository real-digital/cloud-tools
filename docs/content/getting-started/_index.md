---
title: "Getting Started"
date: 2019-10-02T13:52:34+02:00
weight: 3
toc: true
---

#### Introduction
  Ahoi!

  Let me try to introduce you to Pligos again. Please be aware that this
  /getting started/ might seem a little bit fabricated .That's because
  it is. Pligos is meant to support huge environments and it is hard
  to show off Pligos' features on such a simple example. Anyway here
  goes nothing!

  This introduction assumes that you have already familiarized
  yourself with helm and what it does. Pligos is a helm plugin and
  deeply nested with helm, so it might be helpful to have some
  knowledge of helm. If you didn't, try reading and understanding
  [helm.sh/using_helm](https://helm.sh/docs/using_helm/]).

  My take on helm: it's a glorified templating tool that somehow
  managed to check all the right boxes and is therefore extremely
  useful as a kubernetes configuration management tool. On the other
  hand it's shitty for scaling and development, but that's what you
  are here for (hopefully).

#### Flavor or the helm starter with super powers

  Pligos relies on something called a `Flavor`.  You can think of a
  Flavor as a helm starter with super powers. It provides a schema so
  that you actually know what is going on inside of it and isn't
  actually used as a starter for your chart, but as a library of
  template files. As is common with libraries you can share Flavors
  with all your fellow computer nerds without exposing them to your
  service configurations. Also you can collaborate on improving the
  Flavor/library such that everyone benefits from updates!

   A sample flavor named `webservice` is given as following (it is just an example, your flavor could be different) :
   
   ![]({{< resource url="flavor-directory-structure.png" >}})

  So how is a Flavor similar to a helm starter? well it's still a helm
  chart, so let's actually create one using helm. This flavor is going
  to be used by stateless web services. We are going to call it
  webservice!

  ```
  helm create webservice
  ```

  Now, despite this being a chart we will never inject any values into
  it (the reason is that this flavor is just considered to be a template
  and all of our helm charts will be using this template and generating their own
  values.yaml file), so let's remove the `values.yaml`. Tests is also
  something that belongs to your specific chart, so get rid of those as
  well.

  ```
  rm webservice/values.yaml
  rm -r webservice/templates/tests
  ```

  However we do need a `schema.yaml` that defines the schema of the chart.
  So, let's create the `schema.yaml` file inside the `webservice` folder by
  following command:

   
   ```
   echo "context: {values: embedded configuration}" > webservice/schema.yaml
   ```

  And already you got yourself a valid flavor. It's not very useful,
  but we will come to that later.

#### Types

  You might be wondering what the actual hack we just wrote to this
  innocent little schema file. Well at this point even Pligos would
  wonder what it is, because we need to feed you and Pligos more
  information about it! Pligos is operating based on custom
  configuration types. In the example we made use of a
  type called `configuration` and are going to define it now.

  The important part here is, that types are something that YOU
  define. I would love to do that for you, but than I would just force
  on you a shittier version of the kubernetes API because I *can not*
  understand your requirements. That is what Pligos is all about,
  configuring your services in alignment of *your* organization's
  requirements. However I can make some broad guesses in order to
  further kick the tires on this /getting started/. To learn more
  about how to define types and Pligos' DSL go to [schema compiler]({{< ref "/pligos-components/schema/_index.md" >}}).

  Alright, let's define our `configuration` type in the file named
  `types.yaml`. You can create this file anywhere and use its path
  accordingly in the further steps. I've created it outside of the
  `webservice` folder.

  ```
  echo "configuration: { valuesyaml: embedded object }" > types.yaml
  ```

  I am again making use of the Pligos keyword /embedded/ here. Please
  go to the DSL documentation ([schema compiler]({{< ref "/pligos-components/schema/_index.md" >}})) if you want to learn what it does.

  Alright, so we defined our configuration type! Congratulations, not
  that hard, right? It seems that a configuration instance needs to
  define a single key, called `valuesyaml` that needs to be an
  object. I think we can do that! Let's actually start benefiting from
  our efforts and create our first Pligos configuration.

#### Hello-world pligos configuration

  As a helm plugin, every Pligos configuration naturally is also a
  helm chart. We'll create our own helm chart and it will be using the
  `webservice` as the template. I think we have already mastered creating
  helm charts so this one should be easy:

  ```
  helm create hello-world
  ```

  Templates will be provided by our Flavor, so let's kill them with fire!

  ```
  rm -r hello-world/templates
  ```

  In order to morph this beautiful helm chart to a Pligos capable one,
  I propose to replace the `values.yaml` with something more useful.
  Let's remove the `values.yaml` file and create the `pligos.yaml` file
  inside the `hello-world`:

  ```
  rm hello-world/values.yaml
  touch hello-world/pligos.yaml
  ```

  `values.yaml` and our beloved `templates` will come back soon, but,
  as they are generated,  we don't actually need to commit them
  anymore, so I think it's good practice to ignore them:

  ```
  printf "/values.yaml\n/templates\n" > hello-world/.gitignore
  ```

  Let's start editing the `pligos.yaml`. At first we are going to add
  a header to the file that defines some metadata which, most
  importantly links to the types we just created:

  ```
  # hello-world/pligos.yaml

  pligos:
    version: '1'
    types: [../types.yaml]
  ```

  Have you noticed why I have given the path of `types.yaml` as
  "../types.yaml" ?. This is because my `types.yaml` is outside of this
  helm chart folder. But you can give the path of `types.yaml` according
  to the location where you have created it in previous step.

  You might notice, that we can actually plug multiple type definition
  files into your Pligos configuration. Multiple files are simply
  going to be merged.

  We are actually getting really close to something that can be
  deployed, so bare with me, we can do this! The only thing left is
  creating a /Context/.

#### Contexts

  Contexts allow you to manage different versions of your service
  configurations. The most obvious use case is to have a development
  and production version. Doing this without Pligos normally requires
  managing multiple `values.yaml` files, each one overwriting a base
  `values.yaml` file. Using Pligos this nonsense can finally come to
  an end because you can manage all the versions side by side inside
  of one file. You will see how this scales (spoiler: this is what
  type instances are for).

  To ease into the concept let's create a single first context called
  `default` in our `pligos.yaml` which holds the configuration for a
  default helm chart. Go ahead and copy this first context below the
  metadata header we just defined:

  ```
    # hello-world/pligos.yaml

    contexts:
      default:
        flavor: ../webservice
        spec:
          values:
            valuesyaml:
              image:
                repository: nginx
                tag: stable
                pullPolicy: IfNotPresent

              service:
                type: ClusterIP
                port: 80

              ingress: {enabled: false}
  ```

  Looks complicated? Thats ok, because that's not how you would
  normally use Pligos. I only show you this to make clear how simple
  it is to create a configuration instance. As you can see we can now
  use a single context `default` that uses the `webservice` flavor we
  created and references a `configuration` instance using the `values`
  key. Why it uses the `values` key you ask? Well le'ts have a look
  again at our schema definition:

  ```
    # webservice/schema.yaml

    context:
      values: embedded configuration
  ```

  As you can see, the schema we created for the `webservice` Flavor
  requires us to create a `configuration` instance under the `values`
  key, it's as simple as that! And if we think back we can remember,
  that the `valuesyaml` key is part of the `configuration` type.

  Finally done with that, you can now run pligos for the first time
  and will see the `values.yaml` file as well as the templates
  reappear!

  ```
    helm pligos default -c hello-world
    cat hello-world/values.yaml
  ```

  I told you that normally you don't create configurations like
  that. Let's try doing it the Pligos idiomatic way! I suggest that we
  create a /named instance/ of the `configuration` type. Named
  instances are defined under the `values` key which you define at the
  root of the pligos.yaml. Go ahead and copy the following below the
  context definition.

  ```
    # hello-world/pligos.yaml

    values:
      configuration:
        - name: default
          valuesyaml:
            image:
              repository: nginx
              tag: stable
              pullPolicy: IfNotPresent

            service:
              type: ClusterIP
              port: 80

            ingress: {enabled: false}
  ```

  Now change your context definition to the following:

  ```
    # hello-world/pligos.yaml

    contexts:
      default:
        flavor: ../webservice
        spec:
          values: default
  ```

  So, your `pligos.yaml` till this step will look like as following:

  ```

  # hello-world/pligos.yaml

  pligos:
    version: '1'
    types: [../types.yaml]

  contexts:
    default:
      flavor: ../webservice
      spec:
          values: default
  values:
    configuration:
      - name: default
        valuesyaml:
          image:
            repository: nginx
            tag: stable
            pullPolicy: IfNotPresent

          service:
            type: ClusterIP
            port: 80

          ingress: {enabled: false}
  ```

  As you can see, instead of defining configuration instances inline,
  you can created named instances and reference them elsewhere. This
  actually introduces us to the concept of composition within
  Pligos. Go run

  ```
    helm pligos default -c hello-world
    cat hello-world/values.yaml
  ```

  again to make sure nothing on the output changed. BTW you can go
  ahead and deploy this configuration using your familiar helm commands:

  ```
    helm upgrade --install hello-world ./hello-world
  ```

  This should yield the same results as deploying a default helm
  chart without any modifications.

#### Composition

  You made it this far, I am proud of you! We can now finally dive
  into probably the most important feature of Pligos: *composition*.

  Maybe you are like me and use different ingress configurations for
  development and production. For instance, I use a different hostname
  and no tls during development. We could extend our current
  configuration like this to support both environments:

  ```
    # hello-world/pligos.yaml

    contexts:
      development:
        flavor: ../webservice
        spec:
          values: development

      default:
        flavor: ../webservice
        spec:
          values: default

    values:
      configuration:
        - name: development
          valuesyaml:
            image:
              repository: nginx
              tag: stable
              pullPolicy: IfNotPresent

            service:
              type: ClusterIP
              port: 80

            ingress:
              enabled: true
              hosts: [{host: pligos-dev.sh , paths: [ / ]}]

        - name: default
          valuesyaml:
            image:
              repository: nginx
              tag: stable
              pullPolicy: IfNotPresent

            service:
              type: ClusterIP
              port: 80

            ingress:
              enabled: true
              hosts: [{host: pligos.sh , paths: [ / ]}]

              tls:
                - secretName: pligos-tls
                  hosts: [pligos.sh]
  ```

  This does work, however we duplicated a lot of the configuration!
  Wouldn't it be far better to use composition to configure once and
  stick it all together like little legos?

  In order to do this I propose we add an ingress type. Modify your
  `types.yaml` to look like this:

  ```
    # types.yaml

    configuration:
      valuesyaml: embedded object

    tls:
      secretName: string
      hosts: repeated string

    ingress:
      enabled: bool
      hosts: repeated object
      tls: repeated tls
  ```

  As you can see I took the liberty of creating a third type `tls`
  which is not directly used by our context definition, but by the
  ingress type. This shows that composition with Pligos works at any
  level and can be used arbitrarily.

  Let's extend our `schema.yaml` inside of our Flavor to make use of
  the type:

  ```
    # webservice/schema.yaml

    context:
      values: embedded configuration
      ingress: ingress
  ```

  Now let's fix our `pligos.yaml` and free our configuration from all
  the nasty repetition. The end result should look like this:

  ```
    # hello-world/pligos.yaml
    
    pligos:
      version: '1'
      types: [../types.yaml]

    contexts:
      development:
        flavor: ../webservice
        spec:
          values: default
          ingress: development

      default:
        flavor: ../webservice
        spec:
          values: default
          ingress: production

    values:
      tls:
        - name: production
          secretName: pligos-tls
          hosts: [pligos.sh]

      ingress:
        - name: production
          enabled: true
          hosts: [{host: pligos.sh , paths: [ / ]}]
          tls: [production]

        - name: development
          enabled: true
          hosts: [{host: pligos-dev.sh , paths: [ / ]}]

      configuration:
        - name: default
          valuesyaml:
            image:
              repository: nginx
              tag: stable
              pullPolicy: IfNotPresent

            service:
              type: ClusterIP
              port: 80
  ```

  As you can see our configuration looks much cleaner and it is
  immediately obvious where to find what configuration piece and what
  it is used for. We were able to remove the second `configuration`
  instance as the ingress was the only distinguishable factor. Going
  further with this configuration style I can assure you that
  different environments can be defined much more easily and scalable.
  
  So, Congratulations you are done with it. Now, you can again just test to check if everything is working fine by using the Flavor. 
  
  ```
    helm pligos default -c hello-world
    helm upgrade --install hello-world ./hello-world
  ```
  
  
