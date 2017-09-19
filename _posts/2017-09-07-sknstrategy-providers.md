---
layout: post
published: true
title: [SknStrategy - Provider classes]
tagline: Entity Respository Role Containers for SknStrategy
comments: true
---

Responsible for data operations against external sub-systems; like PostgreSQL, Web Services, File Systems, etc.
Performs IO operations on behalf of ServiceDomain methods and strips data package of subsystem related keys or id.
Rails ActiveRecord/ActiveModel APIs are in use in this module. Note: Rails Models are not permitted to include application logic.

![SknStrategy]({{ site.github.repository_url }}/images/SFRegistryMethods.png)

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
# app/strategy/providers/db_profile_provider.rb

module Providers
  class DBProfileProvider < ProvidersBase

    def content_profile_for_runtime(user_profile, available_resource_catalog=true)
      profile = ContentProfile.find_by(person_authentication_key: user_profile.person_authenticated_key).try(:entry_info)
      cprofile = condense_profile_entries(profile, user_profile, available_resource_catalog)
      Rails.logger.debug "#{self.class.name}.#{__method__}() ContentProfile: #{cprofile.present? ? 'Found' : 'Not Found'}"
      cprofile
    end

    def profile_type_select_options_with_description
      ProfileType.option_selects_with_desc
    end

    ...

  end
end
{% endhighlight %}

