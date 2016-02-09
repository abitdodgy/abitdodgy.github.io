---
layout: post
title: Callbacks and Observers
---

A plethora of material exists in the Rails world to dissuade you from using ActiveRecord callbacks. Callbacks are not intrinsically bad, but they give you a lot of rope to hang yourself with, and consequently they're the most abused feature of Rails.

Callbacks make some tasks look and feel deceptively easy, but the honeymoon is soon over, and the pain sets in. It took me about three years of using Rails to be finally convinced that they're best avoided.

Recently, we had to implement an activity log for one of our apps, and callbacks were the first idea that surfaced. In hindsight, it should have been obvious that callbacks are a bad idea for several reasons.

1. An activity log usually involves recording the actions of the current user. Since callbacks are implemented in the model, they have no concept of a current user, nor should they. So I would have to find a way to make the current user available in the model when recording the activity.

2. A model's responsibility should not encompass the recording of activities. It should be aware of its own state and changes to its state, but it should not meddle in the implementation details of the activity log itself.

3. There will be instances when we would appropriate not to record an activity. At the time this seemed unlikely. After all, the purpose of having an activity log was to record changes to the model state, so why would we want to skip that? As it turns out, there will almost always be a reason.

Instead of addressing the problem at the root, I tried to fit square pegs in round holes. I experimented with adding an `attr_accessor` to each object. This was a terrible idea, and I knew it, but I had this perverse desire to keep going until I saw the end result for myself. An `attr_accessor` would allow me to set the current user outside of the model, and have it available when the activity was recorded via a callback.

You may not know it at the time, but there will almost always be a scenario where you would need to disable callbacks. In our case

http://stackoverflow.com/questions/1568218/access-to-current-user-from-within-a-model-in-ruby-on-rails