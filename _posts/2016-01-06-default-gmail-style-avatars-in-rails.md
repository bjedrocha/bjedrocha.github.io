---
layout: 	post
title:		Default Gmail style avatars in Rails
date: 		2016-01-06
summary:	A guide showing how to add Gmail style avatars to a Rails project using the avatar_magick gem.
categories: rails
---

A couple of months ago I released a small gem for generating Gmail style avatars called [avatar_magick](https://rubygems.org/gems/avatar_magick). The gem is a plugin for [Dragonfly](http://markevans.github.io/dragonfly/) and has a couple of different use cases, the most common of which is being able to generate a default avatar in the event that a user doesn't upload their own. This is a common practice in applications with user accounts and it can be handled in different ways. The simplest and probably most commonly used is some form of the head-shoulders silhoutte.

![head-shoulders silhoutte](/images/posts/gmail-style-avatars/default_avatar_silhoutte.png)

This works well in certain cases but is less than ideal in situations where multiple avatars are displayed on a single page (e.g. a comments feed). In such cases, it's best if the avatars have some variation. For example, Github uses indenticons, small pixel sprites that are generated using a hash of the user's ID. Basecamp, on the other hand, will randomly assign one of their [awesome custom painted avatars](https://signalvnoise.com/posts/3104-behind-the-scenes-reinventing-our-default-profile-pictures). Another great approach, and the inspiration behind avatar_magick, is Gmail's use of initial avatars - a single letter representing the sender's name centered on top of a colourful background. This small change can have a positive impact on user experience.

![before-after](/images/posts/gmail-style-avatars/avatar-magick-before-after.gif)

In this guide, I'll cover how to use Dragonfly and the avatar_magick plugin to add this style of initial avatars to an existing Rails project.

For those unfamiliar with [Dragonfly](http://markevans.github.io/dragonfly/), it's a great library for handling images and other attachments but also provides on-the-fly image processing thanks to the included `image_magick` plugin. Dragonfly can be used as an alternative to [paperclip](https://github.com/thoughtbot/paperclip) or used in conjunction to generate on-the-fly thumbnails. By also including the `avatar_magick` plugin, we can extend this functionality further by allowing us to generate initial avatars like those described above.

For the purposes of this guide, the Rails project I'll be working with is a simple application for managing contacts. The fully implemented source is available [here](https://github.com/bjedrocha/avatar-magick-example/tree/dragonfly-attachments). Please use it as a reference.

### Adding and configuring dragonfly and avatar_magick

The first step is adding the `dragonfly` and `avatar_magick` gems to your application. Open your `Gemfile` and add the following before running `bundle install`

{% highlight ruby %}
# Dragonfly and Avatar Magick
gem 'dragonfly', '~> 1.0.11'
gem 'avatar_magick', '~> 1.0.1'
{% endhighlight %}

Next, generate the dragonfly initializer

{% highlight ruby %}
rails g dragonfly
{% endhighlight %}

This will create a `dragonfly.rb` file within the `config/initializers/` directory. Open this file and add the following

{% highlight ruby %}
plugin :avatarmagick
{% endhighlight %}

directly below the `plugin :imagemagick` line. With Dragonfly and avatar_magick configured, let's generate some avatars.

### Avatar Generation

To generate the actual avatars, we'll define a custom endpoint to control the size, background colour, and text of our avatars. Within `routes.rb`, add the following

{% highlight ruby %}
# Avatar routes
get "avatar/:size/:background/:text" => Dragonfly.app.endpoint { |params, app|
  app.generate(:initial_avatar, URI.unescape(params[:text]), { size: params[:size], background_color: params[:background] })
}, as: :avatar
{% endhighlight %}

Now, URIs that match the above route will delegate to Dragonfly which in turn will use the `avatar_magick` plugin to generate an initial avatar. For example, visting `http://localhost:3000/avatar/250/d81b60/bart` in development would produce the following avatar

![avatar](/images/posts/gmail-style-avatars/avatar.png)

### Acts as Avatarable

As mentioned previously, our example Rails application manages contacts. For each contact, we'd like to be able to set the name, email, phone number, and photo. If a photo isn't provided, the application should default to an initial avatar. To handle the bulk of the work, we'll create an `Avatarable` concern which we can then mixin to any model we'd like and override as necessary.

{% highlight ruby %}
# app/models/concerns/avatarable.rb

module Avatarable
  extend ActiveSupport::Concern

  AVATAR_COLORS = [ 'e53935', 'b71c1c', 'd81b60', '880e4f', '8e24aa', '4a148c',
    '5e35b1', '311b92', '3949ab', '1a237e', '1e88e5', '0d47a1', '039be5', '01579b',
    '00acc1', '006064', '00897b', '004d40', '43a047', '1b5e20', '7cb342', '33691e',
    'c0ca33', '827717', 'fdd835', 'f57f17', 'ffb300', 'ff6f00', 'fb8c00', 'e65100',
    'f4511e', 'bf360c', '6d4c41', '3e2723', '757575', '212121', '546e7a', '263238' ]

  included do
    delegate :url_helpers, to: 'Rails.application.routes'
  end

  def avatar_url
    url_helpers.send(:avatar_path, avatar_size, avatar_color, avatar_text)
  end

  def avatar_size
    150
  end

  def avatar_param
    to_param
  end

  def avatar_text
    raise NotImplementedError, "must implement avatar_text"
  end

  def avatar_color
    AVATAR_COLORS[ Zlib.crc32( avatar_param ).modulo( AVATAR_COLORS.length ) ]
  end
end
{% endhighlight %}

Let's step through the above to get a better understanding of how this all works. At the top, we define a constant that holds an array of background colours. While `avatar_magick` works with any colour (for both text and background), I've decided to limit my choices to [Material Design Colours](https://www.google.com/design/spec/style/color.html) that work well with white text. A similar approach can be used if you need to conform to a particular color scheme.

Next we define several methods that are used to construct the paramaters needed to generate the avatar, mainly size, colour, and text. The `avatar_text` method will need to be implemented in whatever class includes this concern and we ensure this by having it raise an exception in the event that it isn't. While the `avatar_size` method is self explanatory, the `avatar_color` method requires a bit of an explanation.

We don't care which colour is chosen for the avatar but we need it to be consistent. In other words, if the initial call to `avatar_color` produces `'43a047'`, subsequent calls should produce this same value. If we simply call `AVATAR_COLORS.sample`, a random _but_ different value will be returned everytime. To ensure consistency, we use the `Zlib` library to calculate the _crc checksum_ of a string returned by `avatar_param` (a stringified version of the model's `id` in our case). The calculated checksum is an integer that will be identical for strings of equal value. Finally, we perform a modulo operation on the checksum to ensure we're in the bounds of our array of colours.

To construct the actual URL for our avatar, we define `avatar_url` which simply delegates to `Rails.application.routes`. Visiting this URL will produce an image like the one shown above.

### Hooking it all up

With our `Avatarable` concern in place, it's time to hook everything up. We'll start by including the concern within our Contact model. This will mixin all the required `avatar_` methods; however, we'll need to implement `avatar_text` as noted above. Our completed Contact model looks like this

{% highlight ruby %}
# app/models/contact.rb

class Contact < ActiveRecord::Base
  extend Dragonfly::Model
  include Avatarable

  dragonfly_accessor :photo

  # associations
  belongs_to :user

  # validations
  validates :first_name, :last_name, :email, presence: true

  def full_name
    [first_name, last_name].join(' ')
  end

  # required for avatarable
  def avatar_text
    first_name.chr
  end
end
{% endhighlight %}

Since we want to show either the uploaded photo or the avatar, we'll create a simple view helper to handle this for us

{% highlight ruby %}
# app/helpers/contacts_helper.rb

module ContactsHelper
  def contact_avatar(contact, options = {})
    if contact.photo.nil?
      image_tag contact.avatar_url, options
    else
      image_tag contact.photo.thumb('150x150#').url, options
    end
  end
end
{% endhighlight %}

We can then use this anywhere we need to display a contact's avatar

{% highlight ruby %}
# app/views/contacts/index.html.erb

<%= contact_avatar(contact, class: 'media-object img-circle', style: 'width:64px;height:64px;') %>
{% endhighlight %}

That's it! We now have great looking default avatars for our contacts along with the ability to upload a custom photo.

![contact-attachments](/images/posts/gmail-style-avatars/contact-photo-attachments.gif)

### Bonus - Paperclip Attachments

If instead of Dragonfly, your app uses [paperclip](https://github.com/thoughtbot/paperclip) to handle image attachments, you can still use `avatar_magick` to provide default avatars. Take a look at the [paperclip-attachments branch](https://github.com/bjedrocha/avatar-magick-example/tree/paperclip-attachments) to see how this is accomplished.

Similarly, if you want to only display the initial avatars and have no need for handling attachments, take a look at the [master branch](https://github.com/bjedrocha/avatar-magick-example/tree/master) to see how to use `avatar_magick` without using Dragonfly's attachment handling feature.