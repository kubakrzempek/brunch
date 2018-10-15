# RSpec - will_be_expected_to

One of the coolest things about RSpec is the ability to put our expectations in form of a one-liner. It is relatively simple, all you have to do is create a subject of the test and make expectation against it.

```ruby
RSpec.describe Array do
  describe "#pop" do
    subject { [1, 2, 3, 4].pop }
    it { is_expected.to eq(4) }
  end
end
```

A suite written in that form is short, concise, and easy to follow.
So far so good. `Array#pop` despite returning the last value also removes the item from the original array, thus lower the size by 1. Let's test that out.

```ruby
RSpec.describe Array do
  subject(:array) { [1, 2, 3, 4] }
  describe "#pop" do
    subject { array.pop }
    it { is_expected.to eq(4) }
    it { is_expected.to change(array, :count).from(4).to(3) }
  end
end
```
Drat! RSpec responded with an error
```
Array#pop should change #count from 4 to 3
     Failure/Error: it { is_expected.to change(array, :count).from(4).to(3)
       expected #count to have changed from 4 to 3, but was not given a block
```

The behavior is kinda obvious since we have to have a block if we want to evaluate post-change effects. It boils down to a rewrite that looks something like that:

```ruby
RSpec.describe Array do
  subject(:array) { [1, 2, 3, 4] }
  describe "#pop" do
    subject { array.pop }
    it { is_expected.to eq(4) }
    it "changes the count from 4 to 3" do
      expect { subject }.to change(array, :count).from(4).to(3)
    end
  end
end
```

which works. And at the same time repeats almost literally our expectation in the description. Also, the "style" of writing starts to diverge (which is not such a big deal, though the one-liners version looks more appealing).

How to get that into one-liner syntax?

```ruby
module SpecHelpers
  module WillBeExpected
    def will_be_expected
      expect { subject }
    end
  end
end

RSpec.configure do |config|
  config.include SpecHelpers::WillBeExpected
end

RSpec.describe Array do
  subject(:array) { [1, 2, 3, 4] }
  describe "#pop" do
    subject { array.pop }
    it { is_expected.to eq(4) }
    it { will_be_expected.to change(array, :count).from(4).to(3) }
  end
end
```

It is not mind-blowing, mind-bending and hackish solution. It comes strictly from a need of having a concise suite. Concise no matter whether we assert return values or post-effects like changing size, raising an error etc. And I believe that every developer admires consistency, especially gained with a low-cost solution.
