---
layout: post
published: true
title: [SknStrategy - ServiceRegistry facility]
tagline: Core Container for SknStrategy
comments: true
---

Responsible for initializing or launching ServiceDomain, Providers, and Processors classes, when invoked by
Rails controller or peer ServiceDomains. ServiceRegistry is a PORO initially instantiated by the
ApplicationController's #method_missing on first use.


![SknStrategy]({{ site.github.repository_url }}/images/SFStrategyModelBasic.png)


{% highlight ruby %}
# app/controllers/application_controller.rb

class ApplicationController < ActionController::Base
  include Registry::RegistryMethods                 # Development Strategy
end
{% endhighlight %}

{% highlight ruby %}
# app/strategy/registry/registry_methods.rb

module Registry
  module RegistryMethods

    # New Services extension
    def service_registry
      @service_registry ||= Services::ServiceRegistry.new({registry: self})
    end

    protected

    # Response wrapper for use in controller
    def wrap_html_response(service_response, redirect_path=root_path)
      @page_controls = service_response
      flash[:notice] = @page_controls.message if @page_controls.message.present?
      redirect_to redirect_path, notice: @page_controls.message and return unless @page_controls.success
    end

    def wrap_html_and_redirect_response(service_response, redirect_path=root_path)
      @page_controls = service_response
      flash[:notice] = @page_controls.message if @page_controls.message.present?
      redirect_to redirect_path, notice: @page_controls.message and return
    end

    def wrap_json_response(service_response)
      @page_controls = service_response
      render(json: @page_controls.to_hash, status: (@page_controls.package.success ? :accepted : :not_found), layout: false, content_type: :json) and return
    end

    # Easier to code than delegation, or forwarder
    def method_missing(method, *args, &block)
      Rails.logger.debug("#{self.class.name}##{__method__}() looking for: #{method.inspect}")
      if service_registry.public_methods.try(:include?, method)
        block_given? ? service_registry.send(method, *args, block) :
            (args.size == 0 ?  service_registry.send(method) : service_registry.send(method, *args))
      else
        super
      end
    end

  end
end
{% endhighlight %}

{% highlight ruby %}
# app/strategy/services/service_registry.rb

module Services
  class ServiceRegistry < ::Registry::RegistryBase

    ##
    # Application Services used in Controller methods
    ##

    def access_service
      @sf_access_service ||= AccessService.new({registry: self})       # First call will execute this set of code
    end

    def content_service
      @sf_content_service ||= ContentService.new({registry: self})
    end

    ##
    # Content Providers
    ##

    def xml_profile_provider
      @sf_xml_profile_builder ||= Providers::XMLProfileProvider.new({registry: self})
    end

    def db_profile_provider
      @sf_db_profile_builder ||= Providers::DBProfileProvider.new({registry: self})
    end

    ##
    # Content Adapters
    ##

    def content_adapter_file_system
      @sf_content_adapter_file_system ||= Processors::FileSystemProcessor.new({registry: self})
    end

    def content_adapter_inline_values
      @sf_content_adapter_inline_values ||= Processors::InlineValuesProcessor.new({registry: self})
    end

    ##
    # Adapter by Content
    # Will accepts ResultBean, Hash, or single string value
    # - An experiment of ways to switch duck targets, not really needed
    def adapter_for_content_profile_entry(content)
      content_type = (content.respond_to?(:to_hash) ? (content[:content_type] || content['content_type']) : content)
      case content_type
        when "Commission", "Activity", "FileDownload", 'Experience', 'FileDownload'
          content_adapter_file_system
        when "Notification", "LicensedStates"
          content_adapter_inline_values
        else
          content_adapter_file_system # default for now
      end
    end

  end
end
{% endhighlight %}

{% highlight ruby %}
# app/strategy/registry/registry_base.rb

module Registry
  class RegistryBase

    attr_accessor :registry

    def initialize(params={})
      params.keys.each do |k|
        instance_variable_set "@#{k.to_s}".to_sym, nil
        instance_variable_set "@#{k.to_s}".to_sym, params[k]
      end
      raise ArgumentError, "#{self.class.name}: Missing required initialization param!" if @registry.nil?
    end

    # User Session Handler
    def get_session_param(key)
      @registry.session[key]
    end

    def set_session_param(key, value)
      @registry.session[key] = value
    end

    # Not required, simply reduces traffic since it is called often
    def current_user
      @current_user ||= registry.current_user
    end

    protected

    # Support the regular respond_to? method by
    # answering for any method the controller actually handles
    #:nodoc:
    def respond_to_missing?(method, incl_private=false)
      registry.send(:respond_to?, method, incl_private) || super
    end


  private

    # Easier to code than delegation, or forwarder; @registry assumed to equal @controller
    def method_missing(method, *args, &block)
      Rails.logger.debug("#{self.class.name}##{__method__}() looking for: #{method}")
      if registry.public_methods.try(:include?, method)
        block_given? ? registry.send(method, *args, block) :
            (args.size == 0 ?  registry.send(method) : registry.send(method, *args))
      else
        super
      end
    end

  end
end
{% endhighlight %}

{% highlight ruby %}
# app/strategy/domains/domains_base.rb

module Domains
  class DomainsBase

    attr_accessor :registry

    def initialize(params={})
      params.keys.each do |k|
        instance_variable_set "@#{k.to_s}".to_sym, nil
        instance_variable_set "@#{k.to_s}".to_sym, params[k]
      end
      raise ArgumentError, "#{self.class.name}: Missing required initialization param!" if @registry.nil?
    end

    ...

  private

    # Easier to code than delegation, or forwarder
    # Allows strategy.domains, service, to access objects in service_registry and/or controller methods by name only: like curent_user
    def method_missing(method, *args, &block)
      Rails.logger.debug("#{self.class.name}##{__method__}() looking for: #{method}")
      block_given? ? registry.send(method, *args, block) :
          (args.size == 0 ?  registry.send(method) : registry.send(method, *args))
    end

  end
end
{% endhighlight %}

{% highlight ruby %}
# app/strategy/providers/providers_base.rb

module Providers
  class ProvidersBase

    attr_accessor :registry

    def initialize(params={})
      params.keys.each do |k|
        instance_variable_set "@#{k.to_s}".to_sym, nil
        instance_variable_set "@#{k.to_s}".to_sym, params[k]
      end
      raise ArgumentError, "#{self.class.name}: Missing required initialization param!" if @registry.nil?
    end

    ...

  private

    # Easier to code than delegation, or forwarder; @registry assumed to equal @controller
    def method_missing(method, *args, &block)
      Rails.logger.debug("#{self.class.name}##{__method__}() looking for: #{method}")
      block_given? ? registry.send(method, *args, block) :
          (args.size == 0 ?  registry.send(method) : registry.send(method, *args))
    end

  end
end
{% endhighlight %}

{% highlight ruby %}
# app/strategy/processors/processors_base.rb

module Processors
  class ProcessorBase

    attr_accessor :registry

    def initialize(params={})
      params.keys.each do |k|
        instance_variable_set "@#{k.to_s}".to_sym, nil
        instance_variable_set "@#{k.to_s}".to_sym, params[k]
      end
      raise ArgumentError, "#{self.class.name}: Missing required initialization param!" if @registry.nil?
    end

    def ready?
      raise NotImplementedError, "#{self.name}##{__method__} Not Implemented!"
    end

    ...

  protected

    # Easier to code than delegation, or forwarder
    # Allows strategy.domains, service, to access objects in service_registry and/or controller by name only
    def method_missing(method, *args, &block)
      Rails.logger.debug("#{self.class.name}##{__method__}() looking for: #{method}")
      block_given? ? registry.send(method, *args, block) :
          (args.size == 0 ?  registry.send(method) : registry.send(method, *args))
    end

  end
end
{% endhighlight %}
---
[back to SknStrategy Introduction]({% link _posts/2017-09-10-sknstrategy-introduction.md %})
{: style="text-align: center;"}
---
