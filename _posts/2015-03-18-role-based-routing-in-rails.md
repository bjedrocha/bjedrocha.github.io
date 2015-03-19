---
layout:     post
title:      Role based routing in Rails
date:       2015-03-18
summary:    Using advanced routing constraints to control output based on user role
categories: rails
---

In a recent project, one of the requirements was to provide different output based on user role. For example, managers (a user role) would be presented with a slightly different UI than operators (another user role). This included things like additional menu items in the layout, additional action links in the view, etc. For the most part this was fairly trivial, I had access to the user object and could use it to make decisions on presentation directly within the view

{% highlight erb %}

<% if current_user.has_role? :manager %>
  <li><%= link_to 'Manage Users' users_path %></li>
<% end %>

{% endhighlight %}

In some cases, the views would be drastically different depending on user role. To avoid littering my views with logic, I decided to create different role-based templates and then wrote a small method that would determine which view template to render

{% highlight ruby %}

# app/controllers/application_controller.rb

private

def role_template
  user_role = current_user.has_role? :manager ? 'manager' : 'operator'
  "#{user_role}_#{action_name}"
end

{% endhighlight %}

I could then use this within any controller that required role based view rendering

{% highlight ruby %}

# app/controllers/orders_controller.rb

# GET /orders/:id
def show
  @order = Order.find params[:id]
  render role_template
end

{% endhighlight %}

The above would render `manager_show.html.erb` if the user was a manager or `operator_show.html.erb` if the user was an operator. So far so good.

### Same route, entirely different content

This is where things got tricky. When logging into the system, users needed to be redirected to their personal dashboards via a common `/dashboard` path and, depending on their role, would need to see entirely different content. In other words, while the URL to access the dashboard would be the same for all users, the data being loaded and presented would be entirely different and dependant on their user role.

A possible approach would have been to use a single controller with a conditional that determined what data to load and which view template to render

{% highlight ruby %}

# app/controllers/dashboards_controller.rb

# GET /dashboard
def show
  if current_user.has_role? :manager
    manager_dashboard
  else
    operator_dashboard
  end
end

private

def manager_dashboard
  # load data required for managers
  render role_template
end

def operator_dashboard
  # load data required for operators
  render role_template
end

{% endhighlight %}

All requests for `/dashboard` could then be routed to this single controller

{% highlight ruby %}

# config/routes.rb

Rails.application.routes.draw do

  get 'dashboard', to: 'dashboards#show'

end

{% endhighlight %}

While this would work, the controller would end up fairly fat, especially if I needed to support more roles in the future. What I really wanted was to use multiple controllers and a way to route the `/dashboard` path to a controller determined by the user role. Enter [Routing Constraints](http://guides.rubyonrails.org/routing.html#advanced-constraints).

Routing constraints are just that - they allow you to define a constraint that the Rails router should use when matching a route. Constraints can be written using Regex or based on any method on the [Request object](http://guides.rubyonrails.org/action_controller_overview.html#the-request-object) that returns a string. For example

{% highlight ruby %}

get 'photos', to: 'photos#index',
  constraints: { subdomain: 'admin' }

{% endhighlight %}

The above constraint would map the `/photos` path to the `PhotosController`'s _index_ action if the requested url contained the _admin_ subdomain (e.g. http://admin.example.com/photos).

For more complex constraints, you can create a ruby class that responds to `matches?` and then pass an initialization of this class as an argument to the `:constraints` option when defining your route

{% highlight ruby %}

class BlacklistConstraint
  def initialize
    @ips = Blacklist.retrieve_ips
  end
 
  def matches?(request)
    @ips.include?(request.remote_ip)
  end
end
 
Rails.application.routes.draw do
  get '*path', to: 'blacklist#index',
    constraints: BlacklistConstraint.new
end

{% endhighlight %}

This is exactly what I was looking for. I created a new folder under `/app` called `constraints` and placed the following into a file called `manager_route_constraint.rb`

{% highlight ruby %}

# app/constraints/manager_route_constraint.rb

class ManagerRouteConstraint
  def matches?(request)
    user = current_user(request)
    user.present? && user.has_role?(:manager)
  end

  def current_user(request)
    User.find_by_id(request.session[:user_id])
  end
end

{% endhighlight %}

The class is very simple, it contains the `matches?` method which looks up the current user (from a session cookie) and checks to see whether that user is a manager. If the user _IS_ a manager, the constraint matches and the route is mapped. If the user _IS NOT_ a manager, the constraint doesn't match and the route is instead mapped to an alternate controller

{% highlight ruby %}

Rails.application.routes.draw do

  constraints ManagerRouteConstraint.new do
    get 'dashboard', to: 'manager_dashboards#show'
  end
  get 'dashboard', to: 'operator_dashboards#show'

end

{% endhighlight %}

This worked well for my particular case but I wanted to make it flexible enough to be re-used with other roles in the future. I renamed the file to `role_route_constraint.rb` and modified it slightly

{% highlight ruby %}

# app/constraints/role_route_constraint.rb

class RoleRouteConstraint
  def initialize(&block)
    @block = block || lambda { |user| true }
  end

  def matches?(request)
    user = current_user(request)
    user.present? && @block.call(user)
  end

  def current_user(request)
    User.find_by_id(request.session[:user_id])
  end
end

{% endhighlight %}

The modified class works in a similar fashion as before but instead of just checking for a particular role, it yields to the block passed into the `initialize` method. This allows it to be used as a constraint for any role

{% highlight ruby %}

Rails.application.routes.draw do

  constraints RoleRouteConstraint.new { |user| user.has_role? :manager } do
    get 'dashboard', to: 'manager_dashboards#show'
  end
  get 'dashboard', to: 'operator_dashboards#show'

end

{% endhighlight %}

With that in place, if a manager requested the `/dashboard` path they would be routed to the _show_ action of the `ManagerDashboardsController` while operators requesting the same path would be routed to the _show_ action of the `OperatorDashboardsController`. The specific controller would then load whatever data was required and present it accordingly.

Overall, I'm very happy with this approach. Constraining a route based on user role allows me to map different controllers to a single path and move the deciding logic to the routing layer. Also, because the data being loaded and presented is so different depending on the user requesting it, having it handled by multiple controllers is much cleaner.