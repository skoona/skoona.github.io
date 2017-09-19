---
layout: post
published: true
title: [Building Applications using the Ruby on Rails Web Framework]
tagline: SknStrategy an implementation alternative to the Rails Way
comments: true
categories: Ruby Rails SknStrategy
---

# Under Construction
Explaining this topic properly will take several articles.  The best source today is the [code repository](https://github.com/skoona/SknServices) and the [Architecture](http://vserv.skoona.net:8080/pages/details_architecture) link in the footer of the code or online demo.


# Introducing SknServices

This [Rails 5 project](https://github.com/skoona/SknServices) is an example of _Authentication and Authorization_ security
processes used with Ruby on Rails(ROR) applications; and a comprehensive example of the **SknStrategy** development method suitable
for building ROR web applications.

![SknStrategy]({{ site.github.repository_url }}/images/SknServices-Strategy.png)

Rails promotes a _MVC-based_ development approach, i.e., The Rails Way.  The MVC pattern is a classic and used with success, as
is, for many applications.  However, there are growth issues when the code set exceeds a certain size or feature mix.  _Big ball
of Mud_, is the current phrase that describes this outcome.

We understand these growth issues and chose to apply a more traditional design bias to implementing applications using
Rails.  We align our structures with _Domain Driven Design(DDD) and Object Oriented Design(OO)_ principals loosely, and do what
is needed to complete the project.

> **ReThink**: _How would you structure your application differently if you choose not to embrace MVC?_

Ordinarily, you would not write your application by embedding it inside a library of any kind, so a rethink of the Rails Way
is not too surprising.  However, Ruby on Rails still Rocks! The **SknStrategy** treats all of Rails as a web library for our
application, and wraps the necessary touch points.

_The Rails MVC Way is the problem, not Rails the Web Framework._

### SknStrategy Impact:

* All ActiveRecord or Active??? API calls are consolidated in Provider classes.
* Business logic is contained in one or more Domain classes.
* All Controller specific APIs are wrapped in Helpers.
* Controllers and Views overall responsibility has been minimized.
* Upgrades from Rails 2 to 3, 3 to 4, and 4 to 5 are typically completed over two days including prep time.
* Adding new features is a straight-forward process! _i.e. No Mud_
* RSpec tests are faster and code-coverage is increased.  We test our entry-points, not Rails.

SknStrategy metaphorically takes one step back from the Rails MVC Way.  Rather than add code to controllers, views, and data models
we implement a classic OO application structure.

![SknFlow]({{ site.github.repository_url }}/images/SFFlow.png)

Rails provides a clean and unobtrusive request/response exchange for your business logic, via Controller entry-point methods.
ActiveRecord, ActiveMailer, and ActiveJob APIs offer the same unobtrusive request/response exchange pattern; if you
structure your code to take advantage of their APIs.

![SknRegistry]({{ site.github.repository_url }}/images/SFRegistryMethods.png)

During the Web request/response cycle all your code will be executing in a Rails Environment.  The SknStrategy uses a short
list of components to add structural support for implementing business logic outside of placing it in Model and Controllers; they are:
* ServiceFactory or ServiceRegistry to instantiate all Business Objects.
* Services and Domain classes grouped by business process, to process the URL request.
* Provider and Processors classes to augment proccess computations, and wrap Active???? apis calls.
* Small collections of Utilities and Helpers to wrap data exchanges with Rails APIs.

SknStrategy makes no demands on what you call various components and is not yet gem'ified.

The first step was to find an alternative to Controller helpers that would instantiate Services.  The ServiceFactory is really just
a list of Controller Helpers gathered into one Class wired back to the Controller via a #before_action and simple #method_missing.  Now asking for
#some_service.some_method in a Controller entry-point is found via a #public_methods lookup on the ServiceFactory instance. (yep:
tried delegate, and forwardable first, #MM was simpler)

Next we reduced the #initialize method params of all services and components to one param, #factory.  A reference to the factory that instantiated it.  This is
accomplished by a common base or parent class for each component groups.

### This is where I put code

```bash
app/
├── assets
├── controllers
│   ├── application_controller.rb
│   ├── pages_controller.rb
│   ├── password_resets_controller.rb
│   ├── profiles_controller.rb
│   ├── sessions_controller.rb
│   ├── user_group_roles_controller.rb
│   └── users_controller.rb
├── helpers
│   └── application_helper.rb
├── jobs
├── mailers
├── models
├── strategy
│   ├── domains
│   │   ├── access_profile_domain.rb
│   │   ├── content_profile_domain.rb
│   │   └── domains_base.rb
│   ├── page_actions_builder.rb
│   ├── processors
│   │   ├── command_base.rb
│   │   ├── file_system_processor.rb
│   │   ├── inline_values_processor.rb
│   │   └── processor_base.rb
│   ├── providers
│   │   ├── db_profile_provider.rb
│   │   ├── providers_base.rb
│   │   └── xml_profile_provider.rb
│   ├── registry
│   │   ├── object_storage_service.rb
│   │   ├── registry_base.rb
│   │   └── registry_methods.rb
│   ├── secure
│   │   ├── access_authentication_methods.rb
│   │   ├── access_registry.rb
│   │   ├── access_registry_utility.rb
│   │   ├── object_storage_container.rb
│   │   ├── user_access_control.rb
│   │   ├── user_profile.rb
│   │   └── user_profile_helper.rb
│   ├── services
│   │   ├── access_service.rb
│   │   ├── content_service.rb
│   │   └── service_registry.rb
│   └── utility
│       ├── build_version.rb
│       ├── content_profile_bean.rb
│       ├── content_profile_test_data_loader.rb
│       ├── development_mail_interceptor.rb
│       └── errors.rb
└── views

```

I started by putting code in the '/lib' like everyone else.  This caused some frustration with namespace not found errors, after ROR upgrade made changes to eager-loading and auto-loading.  I finally reconciled and created the business logic directory under '/app/strategy/'.  This resolved most of the eager-load issues; expecially after I turned off eager-loading.


![ServiceRegistry]({{ site.github.repository_url }}/images/SFStrategyModelBasic.png)

#### To make the connection back to Rails:

Rails Router/Controller entry-point methods must make one call to an application service which will return one bundle of results data, packaged as a ResultsBean.  This results bundle will have at least three keys; #message, #success, and #payload.

Views accept the #payload and expect to find booleans telling it which page elements have been authorized for display and the data as basic ruby values/objects needed to populate on-screen elements.

Several application Helpers have been created to bridge between the application and the controller or view methods.  Helpers enable application services to convert named routes, etc, without having direct knowledge of the controller or ActiveView Rails specific APIs.  This now means that authorization, feature logic, and flow control has to be done is our application through one service method invocation per request.  Shouldn't be a surprise, this is where our code goes.

To enable the controller to find our services a #service_factory method, or service registry method is defined on the ApplicationController along with a #method_missing, implemented to handle delegation.  This ServiceFactory contains the initialization methods for all services, processors, and providers classes in our application.  All SknStrategy components have access to this central ServiceFactory.

The service method invoked from the controllers entry-method is defined in a ProcessService class.  The rethink allows us to gather Domain or Topic level feature methods into one class.  Grouping them by business process should be natural.  AccountService, PolicyService, QuoteService, and ContentService might be names that your application would create as ProcessService class names.

### Service methods:
* Parse the incoming params, collect and package the resulting data into a ResultsBean for the controller/view.
* Always return a proper answer to the controller, which requires it catch any exception thrown by lower level routines.
* Call one domain level method, through inheritance, to generate the response data.

As indicated a ProcessServices class inherits from a ProcessDomain class.  Domain classes have a collection of methods that do the work to produce a valid and pre-authorized response for this application's entry-point.  These methods or commands are likely topical and related to a single business process.  If ActiveRecord/ActiveModel work is required for this process step the domain method will call the appropriate data provider through the service factory to retrieve the business objects needed.  Providers return only business objects; never an ActiveRecord record instance, just attribute bundles.  If part of the process step work requires specialized processing, a TaskAdapter or TaskProccessor class may be invoked to complete the work needed for this request.  Use of standalone PORO single responsibility classes is also valid.  All processed results are returned to the service methods and packaged for the view.

Role-based authorization, Profile-based content authorization, and Warden/Rack-Attach Authentication were also implemented as Rails independent components.  The example application is a minimal shell highlighting a content authorization strategy for protecting downloadable assets.  You will find that regular Rails CRUD is still used in some places, but the majority of the app uses the SknStrategy.

This is how I refactor Rails code.  If this has your interest, examine the code in the GitHub project and try out the online demo.  A PDF with specific guidance for what to expect in the code has been provided.

I'll update this post with more details.  For now, let me know your thoughts in the comments.


James,

### Related Items
* [5 Rails console tricks](http://www.chrisrolle.com/en/blog/5-rails-console-tricks)
* [Piperator: Pipeline Proccessing for Ruby](https://github.com/lautis/piperator)
* [Evented Rails: Decoupling domains in Rails with Wisper pub/sub events](http://www.g9labs.com/2016/06/23/rails-pub-slash-sub-with-wisper-and-sidekiq/)
* [Bring Clarity to your Monolith - Bounded Contexts](https://blog.carbonfive.com/2016/11/01/bring-clarity-to-your-monolith-with-bounded-contexts/)
* [The Rails Way (MVC) ](https://www.sitepoint.com/10-ruby-on-rails-best-practices-3/)
* [Rails MVC Best Practices.](https://www.sitepoint.com/10-ruby-on-rails-best-practices-3/)
* [Clean Architecture.](https://medium.com/magnetis-backstage/clean-architecture-on-rails-e5e82e8cd326)
* [Hexagonal Architecture.](https://medium.com/@vsavkin/hexagonal-architecture-for-rails-developers-8b1fee64a613)
* [DDD for Rails Developers](https://www.sitepoint.com/ddd-for-rails-developers-part-3-aggregates/)
* [Enhanced Ruby on Rails Architecture](https://github.com/CodeRocketCo/enhanced-rails-architecture#enhanced-ruby-on-rails-architecture)
* [Design Patterns in Ruby](https://bogdanvlviv.github.io/posts/ruby/patterns/design-patterns-in-ruby.html#strategy)

### Videos
* [Architecture of The Lost Years talk by Uncle Bob](http://confreaks.com/videos/759-rubymidwest2011-keynote-architecture-the-lost-years)
* [Hexagonal Rails](http://www.slideshare.net/dwhelan/domain-driven-design-and-hexagonal-srchitecture-with-rails)

### Related Books
* [Domain Driven Design, by Eric Evans](https://www.amazon.com/Domain-driven-Design-Tackling-Complexity-Software/dp/0321125215/ref=pd_bxgy_b_img_y)
* [Domain Driven Rails.](https://blog.arkency.com/domain-driven-rails/)
* [Implementing Domain-Driven Design](https://www.amazon.com/gp/product/0321834577/)
* [Event Storming.](https://blog.redelastic.com/corporate-arts-crafts-modelling-reactive-systems-with-event-storming-73c6236f5dd7)
* [Design Patterns in Ruby](https://www.amazon.com/Design-Patterns-Ruby-Addison-Wesley-Professional/dp/0321490452)

End of this post


