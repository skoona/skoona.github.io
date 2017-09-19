---
layout: post
published: true
title: [SknStrategy]
tagline: Development Approach Introduction
comments: true
---

I started with Ruby on Rails around the time version 2.0 was released.  After several release upgrade experiences, I adopted
some different practices to help with future upgrades, increase readability, and to improve code
coverage metrics; as learning's from my prior experiences.

> **_What I learned was; The Rails MVC Way is the problem!, not Rails the Web Framework._**

This post will introduce the components of **_SknStrategy_**, the development approach I
use to build Ruby on Rails applications.

![SknStrategy]({{ site.github.repository_url }}/images/SknServices-Strategy.png)

SknStrategy is a simple structure that can best be characterized as **_"One step back from Rails"_**.   In
that, all business logic has been extracted from Rails container classes (controllers, models, & views) and
inserted into application POROs. Whenever that separation is not practical, we wrapped those Rails APIs in our
own methods to maintain some separation.

Since adopting this development strategy, I have upgraded several corporate applications from Rails
version 2.3 to V4.1 with minimal effort.  Mainly because of the separation of application code from
Rails containers.

> **_The greatest value was derived from removing all application logic/code from the Rails Controllers._**

The business problem is decomposed into related functional grouping or Domains.  Each Domain is responsible
for delivering value related to its namesake.  QuoteDomain, PolicyDomain, or ClaimsDomain, ProductDomain,
ShippingDomain are semantically the types of Domain classes you should expect to create.  However, in the spirit
of **_Ubiquitous Language_**, you should adopt a naming convention that reflects the language used in the business domain.

The SknStrategy is composed of the following five components, all  namespaced under the 'app/strategy' directory:
![SknRegistryMethods]({{ site.github.repository_url }}/images/SFRegistryMethods.png)
<dl>
<dt>ServiceRegistry facility</dt>
<dd>Responsible for initializing ServiceDomain, Providers, and Processors classes, when invoked by Rails
controller or peer ServiceDomains.  ServiceFactory is a PORO initially instantiated by the ApplicationController's
#method_missing on first use.</dd>

<dt>ServiceDomain classes</dt>
<dd>Responsible for validating request params, processing request, and catching any exception thereby garaunteeing
a formatted response to the controller; containing only Ruby values.  A Services class which inherits from a Domain
class is the primary structure; with Services classes taking the request-exception-response role, and the Domain class
taking the processing role.</dd>

<dt>Provider classes</dt>
<dd>Responsible for data operations against external sub-systems; like PostgreSQL, Web Services, File Systems, etc.
Performs IO operations on behalf of ServiceDomain methods and strips data package of subsystem related keys or id.
Rails ActiveRecord/ActiveModel APIs are in use in this module.  Note: Rails Models are not permitted to include application logic.</dd>

<dt>Processor classes</dt>
<dd>Responsible for specialized task processing, as may be needed by ServiceDomain methods.  A method here may instantiate
other PORO classes to complete the complex operations required by a service request.</dd>

<dt>Utility classes</dt>
<dd>A small collection of utilities like NestedHash class that provides dot notation to a hash of key/value pairs.
ServiceDomain methods return their data results inside an instance of this class, as a controller instance variable @page_controls,
to the related controller.  SknUtuls Gem provides the NestedHash class and SknSettings used for application-level configuration values.</dd>
</dl>

These components constitute a framework for application code that is independent of Rails.  How you implement the
business logic inside of the application framework is also unconstrained.  Any Ruby Pattern can be used as needed.

The commonalities observed from a full implementation of SknStrategy include:
* One LOC in the Controller method to invoke the ServiceDomain method!
* One instance var set as the return from the service invocation!
* All Active<?Record> APIs encapsulated in DB-Providers!
* All Rails APIs encapsulated in Controller Helpers!
* Increased clarity on what would be needed to add new pages!
* No longer a need for Rails Controller Tests!

RSpec tests should call every ServiceDomain entry method at least once for the range of possible input params.
ServiceRegistryMockController is a class included in the spec/support directory which fills in for the controller
during testing if needed.  You may be surprised by a code coverage outcome of 80+ percent on average, even though you
did not test controllers.

[SknServices](https://skoona.github.io/SknServices/) source is available on Github as an [online demo](http://vserv.skoona.net:8080/pages/help)

**Source Dir**
```bash
 app/
    ├── assets
    ├── controllers
    │   ├── application_controller.rb
    │   ├── pages_controller.rb
    │   └── profiles_controller.rb
    ├── helpers
    │   └── application_helper.rb
    ├── strategy
    │   ├── domains
    │   │   ├── access_profile_domain.rb
    │   │   ├── content_profile_domain.rb
    │   │   └── domains_base.rb
    │   ├── page_actions_builder.rb
    │   ├── processors
    │   │   ├── file_system_processor.rb
    │   │   ├── inline_values_processor.rb
    │   │   └── processor_base.rb
    │   ├── providers
    │   │   ├── db_profile_provider.rb
    │   │   ├── providers_base.rb
    │   │   └── xml_profile_provider.rb
    │   ├── registry
    │   │   ├── registry_base.rb
    │   │   └── registry_methods.rb
    │   ├── services
    │   │   ├── access_service.rb
    │   │   ├── content_service.rb
    │   │   └── service_registry.rb
    │   └── utility
    └── views
```
**Spec Dir**
```bash
spec/
    ├── rails_helper.rb
    ├── spec_helper.rb
    ├── strategy
    │   ├── domains
    │   │   └── content_profile_domain_spec.rb
    │   ├── page_actions_builder_spec.rb
    │   ├── processors
    │   │   ├── file_system_processor_spec.rb
    │   │   └── inline_values_adapter_spec.rb
    │   ├── providers
    │   │   ├── db_profile_provider_spec.rb
    │   │   └── xml_profile_provider_spec.rb
    │   ├── services
    │   │   ├── access_service_spec.rb
    │   │   └── content_service_spec.rb
    │   └── utility
    └── support
        └── service_registry_mock_controller.rb
```

![SknFlow]({{ site.github.repository_url }}/images/SFFlow.png)

ServiceRegistry implementation details coming in next article.  Until then your comments are welcome.

James,


