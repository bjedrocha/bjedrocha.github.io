---
layout:     post
title:      Role based routing in Rails
date:       
summary:    Using advanced routing constraints to control output based on user role
categories: rails
---

In a recent project, one of the requirements was to provide different output based on user role. For example, managers (a user role) would be presented with a slightly different UI than operators (another user role). This included things like additional menu items in the layout, additional action links in the view, etc. For the most part this was fairly trivial, I had access to the user object and could use it to make decisions on presentation directly within the view

{% highlight erb %}

<% if current_user.has_role? :manager %>
  <li><%= link_to 'Manage Users' users_path %></li>
<% end %>

{% endhighlight %}

In some cases, the views would be drastically different depending on user role. To avoid littering my views with logic, I opted to write a small method that would determine which view template to render

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

This is where things got tricky. When logging into the system, users needed to be redirected to `/dashboard` and, depending on their role, would need to see entirely different content. In other words, not only was the presentation different, but also the data being presented.

I could have used a single controller and a conditional near the top of my action to determine the content to load, then use my `role_template` method to render the appropriate view. This would work but would result in a fat controller which is something I wanted to avoid. What I really wanted was a way to route the `/dashboard` path to a controller determined by the user role. Enter [Routing Constraints](http://guides.rubyonrails.org/routing.html#advanced-constraints).

Routing constraints are just that - they allow you to define a constraint that the Rails router should use when matching a route. Constraints can be written using Regex or based on any method on the [Request object](http://guides.rubyonrails.org/action_controller_overview.html#the-request-object) that returns a string. For more complex constraints, you can create a ruby class that responds to `matches?` and then pass an initialization of this class as an argument to the `:constraints` option when defining your route.

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

The class is very simple, it contains the `matches?` method which looks up the current user and checks to see whether that user is a manager. This worked well for my particular case but I wanted to make it flexible enough to be re-used with other roles. I renamed the file to `role_route_constraint.rb` and modified it slightly

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

With that in place, if a manager requested the `/dashboard` path they would be routed to the _show_ action of the `ManagersController` while operators requesting the same path would be routed to the _show_ action of the `OperatorsController`. The specific controller would then load whatever data was required and present it accordingly. Perfect!