---
layout: post
published: true
title: [SknStrategy - ServiceDomain class]
tagline: Application Logic Container for SknStrategy
comments: true
---

Responsible for validating request params, processing request, and catching any exception
thereby guaranteeing a formatted response to the controller; containing only Ruby values. A
Services class which inherits from a Domain class is the primary structure; with Services classes
taking the request-exception-response role, and the Domain class taking the processing role.

![SknStrategy]({{ site.github.repository_url }}/images/SFRegistryMethods.png)

{% highlight ruby %}
# app/views/profiles/in_action.html.erb

<% provide(:title, accessed_page_name) %>
<article>
  <div class="jumbotron galaxy-dunes_mars">
    <h1 class="big-chalk"><%= accessed_page_name %></h1>
    <p>You have been granted access to certain secure documentation resources.  This page demonstrates
      advanced ContentProfile features used to provide you personalized and secure access to protected resources.</p>
    <p><a class="btn btn-primary btn-lg" href="<%= details_content_pages_path %>" role="button">More about ContentProfile</a></p>
  </div>
  <h3 class="text-center">Welcome <%= @page_controls.payload.cp.display_name %><br/><small>Secured Documents</small></h3>
  <section>
    <div class="panel-group" id="accordion" role="tablist" aria-multiselectable="false" data-accessible-url="<%= @page_controls.payload.get_demo_content_object_url %>">
      <% @page_controls.payload.display_groups.each_with_index do |cpe, idx| -%>
        <div class="col-md-6">
          <div class="panel panel-primary" style="padding-bottom: 6px;" >
            <div class="panel-heading" role="tab" id="heading<%= cpe.topic_type %><%= cpe.content_type %>">
              <h4 class="panel-title">
                <a role="button" data-toggle="collapse" data-parent="#accordion" href="#collapse<%= cpe.topic_type %><%= cpe.content_type %>" aria-expanded="true" aria-controls="collapse<%= cpe.topic_type %><%= cpe.content_type %>">
                 <%= cpe.description %>
                </a>
              </h4>
            </div>
            <div id="collapse<%= cpe.topic_type %><%= cpe.content_type %>" class="panel-collapse collapse <%= 'in' if idx < 2 %>" role="tabpanel" aria-labelledby="heading<%= cpe.topic_type %><%= cpe.content_type %>">
              <div class="panel-body bg-warning" style="max-height:298px; overflow: auto;">
                  <% cpe.content.each do |ctn| -%>
                    <div class="well well-sm col-md-6 runtime-item bg-warning" data-package="<%= ctn.to_json %>"  data-mh="files-group">
                        <div class="text-center">
                          <%= choose_content_icons(ctn) %>
                        </div>
                        <h5 class="text-center"><%= ctn.filename %><br /><small><%= ctn.size %> | <%= ctn.created %></small><br/><small><%= ctn.source %></small></h5>
                    </div>
                  <% end %>
              </div>
            </div>
          </div>
        </div>
      <% end %>
    </div>
  </section>
</article>
{% endhighlight %}

{% highlight ruby %}
# app/helpers/application_controller.rb

module ApplicationHelper

  ...

  # Services helper
  ### Converts named routes to string
  #  Basic '/some/hardcoded/string/path'
  #        '[:named_route_path]'
  #        '[:named_route_path, {options}]'
  #        '[:named_route_path, {options}, '?query_string']'
  #
  # Advanced ==> {engine: :demo,
  #               path: :demo_profiles_path,
  #               options: {id: 111304},
  #               query: '?query_string'
  #              }
  #              {engine: :sym, path: :sym , options: {}, query: ''}
  def page_action_paths(paths)
    case paths
      when Array
        case paths.size
          when 1
            send( paths[0] )
          when 2
            send( paths[0], paths[1] )
          when 3
            rstr = send( paths[0], paths[1] )
            rstr + paths[2]
        end

      when Hash
        rstr = send(paths[:engine]).send(paths[:path], paths.fetch(:options,{}) )
        rstr + paths.fetch(:query, '')

      when String
        paths
    end
  rescue
    '#page_action_error'
  end

  # View helper
  def do_page_actions
    if @page_controls and @page_controls.page_actions?
      PageActionsBuilder.new(@page_controls.hash_from(:page_actions)[:page_actions], self, false).to_s
    end
  end

end # End module
{% endhighlight %}

{% highlight ruby %}
# app/controllers/profiles_controller.rb

class ProfilesController < ApplicationController

  def in_action_admin
    wrap_html_response content_service.handle_in_action_admin(params.to_unsafe_h)
  end

  def api_accessible_content
    wrap_json_response content_service.handle_api_accessible_content(params.to_unsafe_h)
  end

  def member_update
    wrap_html_and_redirect_response content_service.handle_member_updates(params.to_unsafe_h), members_profiles_url
  end

  def api_get_demo_content_object
    @page_controls = content_service.api_get_demo_content_object(params.to_unsafe_h)
    return render( plain: "File not available!", status: :not_found ) unless @page_controls.success and @page_controls.package.package.source?
    send_file(@page_controls.package.package.source, filename: @page_controls.package.package.filename, type: @page_controls.package.package.mime, disposition: :inline) and return
  end

    ...

  end
end
{% endhighlight %}

{% highlight ruby %}
# app/strategy/services/content_service.rb

module Services
  class ContentService < Domains::ContentProfileDomain

    def handle_in_action

      package = in_action_package
      SknUtils::NestedResult.new({
                                     success: package[:cp].present?,
                                     message: package[:message],
                                     payload: package
                                 })
    rescue Exception => e
      Rails.logger.error "#{self.class.name}.#{__method__}() Klass: #{e.class.name}, Cause: #{e.message} #{e.backtrace[0..4]}"
      SknUtils::NestedResult.new({
                                     success: false,
                                     message: e.message,
                                     payload: []
                                 })
    end
    ...

  end
end
{% endhighlight %}

{% highlight ruby %}
# app/strategy/domains/content_profile_domain.rb

module Domains
  class ContentProfileDomain < DomainsBase

    def in_action_package
      profile = db_profile_provider.content_profile_for_runtime(current_user)
      success = profile.present? && profile[:display_groups].present?
      {
        message: (success ? "" : "No Access Provided.  Please contact Customer Service with any questions."),
        cp: (success ? profile : {}),
        display_groups: (success ? profile.delete(:display_groups) : []),
        get_demo_content_object_url: page_action_paths([:api_get_demo_content_object_profiles_path])
      }
    end
    ...

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

    def self.inherited(klass)
      Rails.logger.debug("#{self.name} inherited By #{klass.name}")
    end

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
---
[back to SknStrategy Introduction]({% link _posts/2017-09-10-sknstrategy-introduction.md %})
{: style="text-align: center;"}
---
