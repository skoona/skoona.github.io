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


## ObjectStorageContainer
ObjectStorageContainer is a utility singleton class that retains a reference to complex or expensive objects between requests.  Since,
objects will be garbage collected after every request/response cycle, some type of session-like storage comes in handy.  This is a memory-based
object connected to the ServiceRegistry and Provider classes, it uses the requesting classes name as part of its key/value storage model.

{% highlight ruby %}
# File: app/strategy/registry/object_storage_services.rb
#
# Object Storage Support for domain and service classes
#
module Registry

  module ObjectStorageService

    def self.included(klass)
      klass.extend ClassMethods
      Rails.logger.debug("#{self.name} included By #{klass.name}")
    end

    ##
    # Class Methods
    #
    module ClassMethods
      @@object_storage_service_prefix = nil

      # protected

      # generate a new unique storage key
      def generate_new_storage_key
        object_store.generate_unique_key
      end

      # tests for keys existence
      def query_storage_key?(storage_key)
        object_store.has_storage_key?(storage_key, oscs_get_context)
      end

      # returns stored object
      def retrieve_storage_key(storage_key)
        object_store.get_storage_object(storage_key, oscs_get_context)
      end

      # Saves user object to InMemory Container
      def persist_storage_key(storage_key, obj)
        object_store.add_to_store(storage_key.to_sym, obj, oscs_get_context)
      end

      # Removes saved user object from InMemory Container
      def release_storage_key(storage_key)
        object_store.remove_from_store(storage_key.to_sym, oscs_get_context)
      end

      # Returns number of record in cache
      def count_storage_objects(ctx=nil)
        context = ctx || oscs_get_context
        object_store.size_of_store(context)
      end

      # Purge all over 2 days old
      def purge_older_than_two_days(seconds=nil)
        object_store.purge_by_seconds(seconds) # 2 days is default, else (Time.zone.now - 2.days).to_i
      end

      protected

      def oscs_get_context
        class_variable_get(:@@object_storage_service_prefix)
      end
      def oscs_set_context=(str_val)
        class_variable_set(:@@object_storage_service_prefix, str_val)
      end

      ##
      # Object Storage Container
      # - keeps a reference to hold object in memory between requests
      def object_store
        Secure::ObjectStorageContainer.instance
      end
    end # ClassMethods module


    ##
    # Instance Methods
    #

    # Saves object to inMemory ObjectStore
    # Returns storage key, needed to retrieve
    def create_storage_key_and_store_object(obj)
      key = singleton_class.generate_new_storage_key()
      Rails.logger.debug("#{self.class.name}.#{__method__}(#{obj.class.name}) saved as new with key:#{key}")
      singleton_class.persist_storage_key(key, obj)
      key
    end

    # Updates existing container with new object reference
    # returns object
    def update_storage_object(key, obj)
      Rails.logger.debug("  #{self.class.name}.#{__method__}(#{obj.class.name}) updated existing with key:#{key}")
      singleton_class.persist_storage_key(key, obj)
      obj
    end

    # Retrieves object from InMemory Storage
    # returns object
    def get_storage_object(key)
      obj = singleton_class.retrieve_storage_key(key)
      Rails.logger.debug("  #{self.class.name}.#{__method__}(#{obj.class.name}) retrieved existing with key:#{key}")
      obj
    end

    # Releases object from InMemory Storage
    # returns object, if present
    def delete_storage_object(key)
      obj = singleton_class.release_storage_key(key)
      Rails.logger.debug("#{self.class.name}.#{__method__}(#{obj.class.name}) removed existing object with key:#{key}")
      obj
    end

    # Checks if object is in Storage
    # returns true|false
    def is_object_stored?(key)
      rc = singleton_class.query_storage_key?(key)
      Rails.logger.debug("#{self.class.name}.#{__method__}() existing object with key:#{key}, exists?:#{rc ? 'True' : 'False'}")
      rc
    end

    def purge_storage_objects(seconds=nil)
      rc = singleton_class.purge_older_than_two_days(seconds)
      Rails.logger.debug("#{self.class.name}.#{__method__}() purged #{rc} items from storage.")
      rc
    end

  end # end ObjectStorageService namespace

end # end registry namespace

{% endhighlight %}

{% highlight ruby %}
##
# <Rails.root>/lib/Secure/object_storage_container.rb
#
## Stores Objects in memory using ThreadSafe methods
#  - keys are expected to be unique and greater than 16 bytes long or (UUIDs)
#    * generate_new_key() produces a 32 char UUID
#  - context is a prefix for keys, which allows multiple caches in same storage container
#  - The internal storage template is: {key: [object, timestamp]}
#    * The timestamp allows for later cleanup if needed
#
# -- Maybe and Alternate Utility
# Ref: http://guides.rubyonrails.org/caching_with_rails.html#activesupport-cache-store

module Secure
  class ObjectStorageContainer
    include Singleton

    COVERRIDE = "Admin"  # Context Override
    CDEFAULT =  "Warden"

    def initialize
      Rails.logger.debug("Secure::ObjectStorageContainer => #{self.class.name} initialized.")
      @objects_storage_container = Hash.new
    end

    def generate_unique_key
      SecureRandom.hex(16)    # returns a 32 char string
    end
    def get_new_secure_digest(token)
      BCrypt::Password.create(token, cost: (BCrypt::Engine::MIN_COST + SknSettings.security.extra_digest_strength)) # Good and Strong
    end

    def remove_from_store(key, context=CDEFAULT)
      store_key = "#{context}.#{key.to_s}".to_sym
      rc = @objects_storage_container.key?(store_key) ? @objects_storage_container.delete(store_key).first : nil
      Rails.logger.debug "#{self.class.name}.#{__method__}(#{context}) Key=#{store_key.to_s}"
      rc
    end

    # Remove all entries more than two days old
    # Returns count removed
    def purge_by_seconds(seconds=nil)
      counter = 0
      expired = seconds || (Time.zone.now - 2.days).to_i # as an integer number of seconds since the Epoch.
      @objects_storage_container.delete_if do |k,v|
        if v.last < expired
          counter += 1
          true
        else
          false
        end
      end
      Rails.logger.perf "#{self.class.name}.#{__method__}() Count=#{counter}"
      counter
    end

    def add_to_store(key, object, context=CDEFAULT)
      store_key = "#{context}.#{key.to_s}".to_sym
      @objects_storage_container.update({store_key => [object, Time.zone.now.to_i]})
      Rails.logger.debug "  #{self.class.name}.#{__method__}(#{context}) Key=#{store_key.to_s}"
      true # prevent return of full hash
    end

    def get_storage_object(key, context=CDEFAULT)
      store_key = "#{context}.#{key.to_s}".to_sym
      rc = @objects_storage_container.key?(store_key) ? @objects_storage_container[store_key].first : nil
      Rails.logger.debug "  #{self.class.name}.#{__method__}(#{context}) Key=#{store_key.to_s}"
      rc
    end

    def has_storage_key?(key, context=CDEFAULT)
      store_key = "#{context}.#{key.to_s}".to_sym
      @objects_storage_container.key?(store_key)
    end

    def size_of_store(context=CDEFAULT)
      counter = 0
      @objects_storage_container.each_key do |k|
        if context.eql?(COVERRIDE)
          counter += 1
        elsif k.to_s.starts_with?(context)
          counter += 1
        end
      end
      Rails.logger.debug "  #{self.class.name}.#{__method__}(#{context}) KeyCount=#{counter}"
      counter
    end

    #
    # Returns array of arrays where [[key-without-context, username],...]
    def list_storage_keys_and_value_class()
      results = []
      @objects_storage_container.each_pair do |k,v|
        name = v.first.try(:username) || v.first.try(:[], :username) || "not found"
        parts = k.to_s.split('.')
          results << {
              klass: parts[0],
              key: parts[1],
              vklass: v.first.class.name,
              ref: name,
              time: Time.at(v.last).strftime("%Y-%m-%d %H:%M:%S %p")
          }
      end
      Rails.logger.debug "#{self.class.name}.#{__method__}() Results=#{results.count}"
      results
    end

    ##
    # Test Support -- Clear without delay
    def test_reset!
      @objects_storage_container.clear
      true
    end

    ##
    # Marshalling Support to preserve state of objects_storage_container
    #
    def _dump(depth=-1)
      Marshal.dump(@objects_storage_container, depth)
    end

    def self._load(str)
      instance.reload_store Marshal.load(str)
      instance
    end

    def reload_store(store)
      @objects_storage_container = store
    end
  end
end

{% endhighlight %}
---
[back to SknStrategy Introduction]({% link _posts/2017-09-10-sknstrategy-introduction.md %})
{: style="text-align: center;"}
---
