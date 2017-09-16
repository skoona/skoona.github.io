---
layout: post
published: true
title: [Building Applications using the Ruby Rails Web Framework]
tagline: Building Applications using the Ruby on Rails Web Framework
comments: true
---
## Under Construction

![ServiceRegistry]({{ site.github.repository_url }}/images/SFStrategyModelBasic.png)

Introducing SknServices
This GitHub hosted Rails 5 project, https://github.com/skoona/SknServices, is a small example of core authentication and authorization security processes; and a real example of an alternative development strategy for Rails web applications.

Rails promotes a MVC-based development approach, i.e., The Rails Way.  The MVC pattern is a classic and used with success, as is, for many applications.  However, there are growth issues when the code set exceeds a certain size or feature mix.  Big ball of mud, is the current phrase that describes this outcome.  We understand these growth issues and have chosen to apply a more traditional design bias to implementing applications using Rails.  The bias here means to align with DDD and OO principals loosely, but we do what is needed to get the job done.

The SknStrategy involves treating all of Rails as a web library for our application.  Ordinarily, you would not write your application by embedding it inside a library of any kind, so a rethink of the Rails Way is required.

SknStrategy Impact:

All ActiveRecord or Active??? API calls are consolidated in a Provider class.

Business logic is contained in one or more Domain classes.

All Controller specific APIs are in Helpers.

Controllers and Views overall responsibility is greatly reduced.

ReThink: How would you structure your application differently if you choose not to embrace MVC?

SknStrategy metaphorically takes one step back from the Rails MVC Way.  Rather than add code to controllers, views, and data model we implement a classic OO application structure; right now we refer to it as the SknStrategy.

To make the connection back to Rails:

Rails Router/Controller entry-point methods must make one call to an application service which will return one bundle of results data, packaged as a ResultsBean.  This results bundle will have at least three keys; #message, #success, and #payload.

Views accept the #payload and expect to find booleans telling it which page elements have been authorized for display and the data as basic ruby values/objects needed to populate on-screen elements.

Several application Helpers have been created to bridge between the application and the controller or view methods.  Helpers enable application services to convert named routes, etc, without having direct knowledge of the controller or ActiveView Rails specific APIs.  This now means that authorization, feature logic, and flow control has to be done is our application through one service method invocation per request.  Shouldn't be a surprise, this is where our code goes.

To enable the controller to find our services a #service_factory method, or service registry method is defined on the ApplicationController along with a #method_missing, implemented to handle delegation.  This ServiceFactory contains the initialization methods for all services, processors, and providers classes in our application.  All SknStrategy components have access to this central ServiceFactory.

The service method invoked from the controllers entry-method is defined in a ProcessService class.  The rethink allows us to gather Domain or Topic level feature methods into one class.  Grouping them by business process should be natural.  AccountService, PolicyService, QuoteService, and ContentService might be names that your application would create as ProcessService class names.

Service methods:

Parse the incoming params, collect and package the resulting data into a ResultsBean for the controller/view.

Always return a proper answer to the controller, which requires it catch any exception thrown by lower level routines.

Call one domain level method, through inheritance, to generate the response data.

As indicated a ProcessServices class inherits from a ProcessDomain class.  Domain classes have a collection of methods that do the work to produce a valid and pre-authorized response for this application's entry-point.  These methods or commands are likely topical and related to a single business process.  If ActiveRecord/ActiveModel work is required for this process step the domain method will call the appropriate data provider through the service factory to retrieve the business objects needed.  Providers return only business objects; never an ActiveRecord record instance, just attribute bundles.  If part of the process step work requires specialized processing, a TaskAdapter or TaskProccessor class may be invoked to complete the work needed for this request.  Use of standalone PORO single responsibility classes is also valid.  All processed results are returned to the service methods and packaged for the view.

Role-based authorization, Profile-based content authorization, and Warden/Rack-Attach Authentication were also implemented as Rails independent components.  The example application is a minimal shell highlighting a content authorization strategy for protecting downloadable assets.  You will find that regular Rails CRUD is still used in some places, but the majority of the app uses the SknStrategy.

This is how I refactor Rails code.  If this has your interest, examine the code in the GitHub project and try out the online demo.  A PDF with specific guidance for what to expect in the code has been provided.

I'll update this post with more details.  For now, let me know your thoughts in the comments.


James,


* [The Rails Way (MVC) ](https://www.sitepoint.com/10-ruby-on-rails-best-practices-3/)
* [Rails MVC Best Practices.](https://www.sitepoint.com/10-ruby-on-rails-best-practices-3/)
* [SknStrategy.](https://skoona.blogspot.com/2016/08/sknservices-alternate-development_11.html)
* [Clean Architecture.](https://medium.com/magnetis-backstage/clean-architecture-on-rails-e5e82e8cd326)
* [Hexagonal Architecture.](https://medium.com/@vsavkin/hexagonal-architecture-for-rails-developers-8b1fee64a613)
* [Domain Driven Design for Rails.](https://blog.arkency.com/domain-driven-rails/)
* [Event Storming.](https://blog.redelastic.com/corporate-arts-crafts-modelling-reactive-systems-with-event-storming-73c6236f5dd7)


End of this post


