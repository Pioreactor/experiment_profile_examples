Experiment profiles are a way to automate your Pioreactor's actions from a flat YAML file. 

The `if` directive can be included in any action to conditionally execute it or not. The `if` statement is evaluated _when the action is to be executed_ (i.e., when `elapsed_hours` has passed).

The `if` directive is a general boolean expression, and to common operators can be used:
 - `and`,`or`,`not`, True, False,`(` and `)`

Also included are numbers, and strings. The comparison operators are are the usual operators.

The operators addition `+`, `-`, `*`, and `/` are allowed on floats, as well. The power of `if` comes when you combine it with expressions, see below:

Expressions are the method used to get is dynamic data, provided from jobs, during execution of profiles. For example, the following:

``
pio1::stirring::target_rpm
``

will fetch the `target_rpm` from `pio1`'s `stirring` job at the time the action is to be executed. To use this in an example:

``
    stirring:
     ...
     - type: update
       hours_elapsed: 6.0
       if: pio1::stirring::target_rpm >= 500
       options:
         target_rpm: 400

``

will check, after 6 hours, if the `target_rpm` is above 500, and if true, will update the target RPM to 400.

You can also compare against strings. For example, to stop a job if the temperature automation running is equal to `thermostat`, use:

``
    temperature_control:
     ...
     - type: stop
       hours_elapsed: 6.0
       if: pio1::temperature_control::automation_name == thermostat
``

Where do these dynamic values come from? Each job has `published_settings` that can be referenced (refer to the job's source code to all `published_settings` for a job, or they are published in MQTT).

Some published settings have are actually nested json blobs, but we need either numbers or strings to compare in our boolean expression. You can index these json blobs in the boolean expression using `.`, for example:

``
    temperature_automation:
     ...
     - type: update
       hours_elapsed: 6.0
       if: pio1::temperature_control::temperature.temperature <= 30
       options:
         target_temperature: 32
``


We use `temperature.temperature` because the `temperature` published setting is a json blob that looks like the following, and we wish to reference the "temperature" field in the blob:
``
    {
        "temperature": <float>,
        "timestamp": <8601 timestamp>
    }
``

Similar to an `if` directive using dynamic data, options can also have dynamic data (see notes above for syntax, too). However, to distinguish between a string and an expression, an expression must be wrapped in `${{ ... }}`. For example, consider the following `update` action:

``
pioreactors:
  worker1:
    jobs:
      stirring:
        actions:
          - type: start
            hours_elapsed: 0
            options:
              target_rpm: 500
          - type: update
            hours_elapsed: 12
            options:
              target_rpm: ${{ worker1:stirring:target_rpm + 50 }}

``

This will update the value of `target_rpm` to whatever its current value is (after 1 hour), and add 50 to it.

You can use the any pioreactor and any job in an expression - you aren't limited to the `job` your editing. For example, the `update` below will dynamically set the `target_rpm` to a function of optical density.

``
pioreactors:
  worker1:
    jobs:
      stirring:
        actions:
          - type: start
            hours_elapsed: 0
            options:
              target_rpm: 500
          - type: update
            hours_elapsed: 12
            options:
              target_rpm: ${{ worker1:stirring:target_rpm + worker1:od_reading:od1.od * 10 }}
``

Expressions can reference individual Pioreactors, for example `worker1:stirring:target_rpm`, but what if you want to specify all Pioreactors in an expression? This is useful for using expressions in the `common` block. The syntax for this is to use the following

``
::<job_name>:setting
``

For example, to conditionally change the stirring RPM in all Pioreactors, and to update it:

``
common:
  jobs:
    stirring:
      actions:
        - type: update
          hours_elapsed: 6
          if: ${{ ::stirring:target_rpm <= 500 }}
          options:
            target_rpm: 500
``

You can also use this syntax in `options`:

``
common:
  jobs:
    stirring:
      actions:
        - type: update
          hours_elapsed: 6
          if: ${{ ::stirring:target_rpm <= 500 }}
          options:
            target_rpm: ${{ ::stirring:target_rpm + 10 * ::od_reading:od1.od }}
``

The `repeat` directive is the most powerful action, as it allows you loop actions over and over again to check for a condition change, update based on state, etc.

The `repeat` action requires two  necessary fields:

 - `actions`: a list of actions (`start`, `stop`, `update`, etc.) that you want to repeat. The field `hours_elapsed` refers to the start of the loop, _not_ when the profile starts.
 - `repeat_every_hours`: this is a float describing how long, in hours, the loop should last for. For example,  repeat an action every 2 hours, or generally: repeat a sequence of actions every X hours.

There is more control using the other optional fields:

 - `max_hours`: this controls how long the loop should run for. For example, if `repeat_every_hours` is `0.5` (or 30 minutes), and `max_hours` is `6`, then the loop will repeat 12 times before exiting.
 - `while`: this is an expression, like `if`, that runs at the start of each loop, including the first. For example, the following profile will run media until the OD is less than 3.0. We also remove waste so we don't overflow the vial. This is a really coarse turbidostat, and is just for demonstration - don't use this:
 - You can also use the `if` directive to skip running the entire `repeat` action, too.


A common task is to start the thermostat, and you may want to do something like:
``
# wrong, this won't work  ❌
common:
  jobs:
    thermostat:
      actions:
        - type: start
          hours_elapsed: 0.0
          options:
            target_temperature: 30

``
However, to start (and stop) automations (which the `thermostat` is), you actually need to use the controllers:

``
common:
  jobs:
    temperature_control:
      actions:
        - type: start
          hours_elapsed: 0.0
          options:
            automation_name: thermostat
            target_temperature: 30

``

Likewise, to stop a automation, you need to stop the controller:

``
common:
  jobs:
    temperature_control:
      actions:
        - type: start
          hours_elapsed: 0.0
          options:
            automation_name: thermostat
            target_temperature: 30
        - type: stop
          hours_elapsed: 12.0

``

A setting like `target_temperature` or `volume` is attached to the automation, not a controller, so you need to update the automation:

``
common:
  jobs:
    temperature_control:
      actions:
        - type: start
          hours_elapsed: 0.0
          options:
            automation_name: thermostat
            target_temperature: 30
        - type: stop
          hours_elapsed: 12.0
    temperature_automation: # this is thermostat
        - type: update
          hours_elapsed: 1.0
          options:
            target_temperature: 32
        - type: update
          hours_elapsed: 2.0
          options:
            target_temperature: 34
``