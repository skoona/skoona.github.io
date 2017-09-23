---
layout: post
published: true
title: [SknStrategy - Processor class]
tagline: Special Task Processor Container for SknStrategy
comments: true
---

Responsible for specialized task processing, as may be needed by ServiceDomain methods. A method here may instantiate
other PORO classes to complete the complex operations required by a service request.

![SknStrategy]({{ site.github.repository_url }}/images/SFFlow.png)

{% highlight ruby %}
# app/strategy/domains/content_profile_domain.rb

module Domains
  class ContentProfileDomain < DomainsBase

    def get_content_object_api(params)
       adapter_for_content_profile_entry(params).retrieve_content_object(params)
    end

    ...

  end
end
{% endhighlight %}

{% highlight ruby %}
# app/strategy/processors/file_system_processor.rb

module Processors
  class FileSystemProcessor < ProcessorBase

    def initialize(params={})
      super(params)
      @file_system = Pathname('./controlled/projects')
    end

    # {"id"=>"0:0:1", "username"=>"developer", content_type="Commission" "controller"=>"profiles", "action"=>"api_get_content_object"}
    def retrieve_content_object(params, user_p=nil) # Hash entry result from available_content_list method
      page_user = user_p || (get_page_user(params["username"] || params[:username]))

      catalog = get_storage_object("#{PREFIX_CATALOG}-#{page_user.person_authenticated_key}")
      result = {
          success: true,
          package: catalog.try(:[], params[:id]) || {}              # { source:, filename: , mime: }  key should be :id but prior method flipped value to :profile
      }
      Rails.logger.debug "#{self.class}##{__method__}() Catalog: #{catalog.keys}, Result: #{result.present?}"

      result

    rescue Exception => e
      Rails.logger.warn "#{self.class.name}.#{__method__}(catalog: #{catalog}) Klass: #{e.class.name}, Cause: #{e.message} #{e.backtrace[0..4]}"
      {
          success: false,
          package: {}
      }
    end

    ...

  end
end
{% endhighlight %}
---
[back to SknStrategy Introduction]({% link _posts/2017-09-10-sknstrategy-introduction.md %})
{: style="text-align: center;"}
---
