# How to test concerns in Rails controller

Very short and quick introduction via commented test file.

```ruby
# test/controllers/concerns/my_concern_test.rb
require "test_helper"

# First define concerned controller to use in the tests
class DeeplyConcernedController < ApplicationController
  include MyConcern

  def action
    render "works", status: :ok
  end
end

class MyConcernTest < ActionDispatch::IntegrationTest
  setup do
    # Add the route to the application...
    Rails.application.routes.draw do
      get "/action" => "deeply_concerned#action"
    end
  end

  teardown do
    # ... and restore originals in the teardown
    Rails.application.reload_routes!
  end

  test "setting organization scope around controller's action" do
    get "/action"
    # Test your concern's side effects here, e.g. setting up a value in current thread,
    # updates in DB, you name it.
    assert_response :ok
    assert_equal "works", response.body
  end
end
```
