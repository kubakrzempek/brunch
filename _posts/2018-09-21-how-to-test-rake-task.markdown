# How to test a rake task

Annotated code below. Even though the example is tied to Rails it should be pretty straightforward to copy to other testing tools (RSpec, plan Minitest, you name it).

```ruby
require "test_helper"
require "rake"

class MyRakeTaskTest < ActiveSupport::TestCase
  setup do
    # By requiring rake we can use its helper to load the file with the tasks
    # we want to test. Another way to load it would be via `load` as follows
    # 
    # load File.expand_path("../lib/tasks/my_rake_task.rake", __FILE__)
    # 
    # For Rails users
    # Rails.application.load_rake_tasks
    # would also work, but that is a heavy-weight operation and I'd avoid that.
    Rake.application.rake_require "tasks/my_rake_task"

    # If your task depends on other tasks you should load them as well. In Rails it is typical
    # to depend on `environment` (which happens to be loaded in test already, we are in some env
    # after all), so defining a "mock" is more than enough.
    Rake::Task.define_task(:environment)
    
    # reenable - without that, you can't run the task more than once. I mean you can, but it won't
    # be reevaluated, and you'll be served with cached content.
    Rake::Task["my_rake_task"].reenable
  end

  test "very important side effect" do
    Rake::Task["my_rake_task"].invoke
    # Happy testing!
  end
end
```
