---
layout: post
published: true
title: [SknStrategy - Utility classes]
tagline: Ruby Key/Value Container for SknStrategy
comments: true
---

A small collection of utilities like NestedHash class that provides dot notation to a hash of key/value pairs.
ServiceDomain methods return their data results inside an instance of this class, as a controller instance variable
@page_controls, to the related controller. [SknUtils](https://skoona.github.io/skn_utils/) Gem provides the NestedHash
class and SknSettings used for application-level configuration values.

![SknStrategy]({{ site.github.repository_url }}/images/SknServices-SknStrategy.png)


{% highlight ruby %}
# app/controllers/profiles_controller.rb

class ProfilesController < ApplicationController

  def in_action
    @page_controls = content_service.handle_in_action
    redirect_to root_path, notice: @page_controls.message and return unless @page_controls.success
    flash[:notice] = @page_controls.message if @page_controls.message.present?
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
# app/helpers/application_controller.rb

module ApplicationHelper

  ...

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

  def do_page_actions
    if @page_controls and @page_controls.page_actions?
      PageActionsBuilder.new(@page_controls.hash_from(:page_actions)[:page_actions], self, false).to_s
    end
  end

end # End module
{% endhighlight %}


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
