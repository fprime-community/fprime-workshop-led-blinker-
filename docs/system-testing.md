# System Testing

In this section, we will walk through the creation of system tests, also known as integration tests, for the LED component created in prior steps.

F´ system tests use a Python api to dispatch commands to a deployment using the F´ GDS, verifying components behave as expected as part of deployment running on actual hardware.

Before starting this guide, users should have the LedBlinking deployment running on their hardware and connected to the F´ GDS running on a development machine. If hardware is not available, this guide can be followed by running the LedBlinking deployment locally on a development machine instead.

> Note: If running the LedBlinker deployment locally instead of on the intended hardware, make sure to rebuild F´ with stubbed GPIO drivers so the LedBlinker deployment doesn't attempt to write to physical GPIO ports. Regenerate the native deployment with `fprime-util generate -DFPRIME_USE_STUBBED_DRIVERS=ON`. MacOS defaults to stubbed drivers and does not require explicitly setting this option.

## Intro to F´ System Testing

Installing the fprime-gds also installs a pytest fixture called `fprime_test_api`. When used, this allows the pytest testing library to automatically connect to the currently running fprime-gds.

> Note: if running the F´ GDS on non-default ports, you can use the same command line arguments used with `fprime-cli` with `pytest` to point the system testing library to the correct GDS instance

First, create a basic test case to verify the system testing library is correctly setup.


Open the file `Components/Led/test/int/led_integration_tests.py` and add the following contents:

```python
def test_cmd_no_op(fprime_test_api):
    """Test command CMD_NO_OP

    Test that CMD_NO_OP can be sent and return without and errors
    """
    fprime_test_api.send_and_assert_command("LedBlinker.cmdDisp.CMD_NO_OP")
```

This test will send a `CMD_NO_OP` command and verify if successfully returns.

Next, after verifying the F` GDS is connected to your deployment, run the new system test and confirm it executes successfully.

```shell
# In led-blinker/LedBlinker
# Requires fprime-gds to be running and connected to LedBlinker deployment
$ pytest ../Components/Led/test/int/led_integration_tests.py
```

## F´ System Testing Assertions

The typical pattern for F´ system tests is to send a command, then wait until some condition is met, such as receiving an event. If the system test don't receive the expected results, they will time out and fail.

`fprime_test_api.send_and_assert_command` will send a command and wait for its completion, but there are several other variants we will use in this guide.

For example, `fprime_test_api.send_and_assert_event` will send a command, then wait for specific events to be emitted.

Independently from sending commands, you can also search the history of received of events and telemetry to make sure deployments are sending the expected data.

There are several variants of these assertions, but this guide will use `fprime_test_api.assert_telemetry` is used to check the expected telemetry values arrive and `fprime_test_api.assert_telemetry_count` to assert that the expected amount of telemetry arrives.

## Testing SetBlinking On/Off

Next, we will develop a test for the `SetBlinkingState` command.


Add the following to your previously created system test `Components/Led/test/int/led_integration_tests.py`:


```python
import time

from fprime_gds.common.testing_fw import predicates

def test_blinking(fprime_test_api):
    """Test that LED component can respond to ground commands"""
```

First, we will turn blinking on, verify that we receive a `SetBlinkingState` event, then check that `LedState` on and off events start arriving. After, we will turn blinking off, and make sure that a `SetBlinkingState` off event arrives.

Add the following to the `test_blinking()` method:

```python
    # Send command to enable blinking, then assert expected events are emitted
    blink_start_evr = fprime_test_api.get_event_pred("LedBlinker.led.SetBlinkingState", ["ON"])
    led_on_evr = fprime_test_api.get_event_pred("LedBlinker.led.LedState", ["ON"])
    led_off_evr = fprime_test_api.get_event_pred("LedBlinker.led.LedState", ["OFF"])

    fprime_test_api.send_and_assert_event(
        "LedBlinker.led.BLINKING_ON_OFF",
        args=["ON"],
        events=[blink_start_evr, led_on_evr, led_off_evr, led_on_evr],
    )

    # Send command to stop blinking, then assert blinking stops
    #TODO: Define blink_stop_evr
    fprime_test_api.send_and_assert_event(
        "LedBlinker.led.BLINKING_ON_OFF", args=["OFF"], events=[blink_stop_evr]
    )
```

### Do it Yourself

Below is a table with a task you must complete before moving on to the next section. This task require you to update the section of code you just added.

| Task | Solution |
|-------|-------------|
| 1. Add event pred definition for use in `fprime_test_api.send_and_assert_event("LedBlinker.led.BLINKING_ON_OFF", args=["OFF"], events=[blink_stop_evr])` | <details><summary>Answer</summary>`blink_stop_evr = fprime_test_api.get_event_pred("LedBlinker.led.SetBlinkingState", ["OFF"])`</details> |

Now, run `pytest` and verify the new test passes. Open the F` GDS webpage and see if the expected events from the test appear in the event viewer.

## Test BlinkingState Telemetry

Next, we will check that the BlinkingState telemetry channel is set ON and OFF when setting blinking on and off.

`fprime_test_api.assert_telemetry` can be used to check that telemetry matches expectations.

In `test_blinking()`, after turning blinking on, add the following the check that BlinkingState is set ON.

```python
    # Assert that blink command sets blinking state on
    blink_state_on_tlm = fprime_test_api.get_telemetry_pred("LedBlinker.led.BlinkingState", "ON")
    fprime_test_api.assert_telemetry(blink_state_on_tlm)
```

Run `pytest` and make sure the new assertion passes.

Now, add the similar assertion to verify the BlinkingState is set OFF after stopping blinking
Run `pytest` again. **Notice that this new telemetry check should fail.**

```python
# Assert that blink command sets blinking state off
#TODO: use fprime_test_api.assert_telemetry to check that "LedBlinker.led.BlinkingState" is off
```

Events in F` are emitted immediately, but telemetry is only emitted periodically. In the LedBlinker deployment, telemetry channels are sent once per second.

The `fprime_test_api.assert_telemetry` check will immediately search for a matching `LedBlinker.led.BlinkingState` telemetry value.
However, because one second hasn't passed between setting blinking off and checking for telemetry, there hasn't been sufficient time for the updated telemetry value to be sent.

To correct this, add `timeout=2` to `fprime_test_api.assert_telemetry`. This will allow this check to wait for up to 2 seconds to receive the expected telemetry before failing.

Run `pytest` and the tests should now pass.

## Advanced: Test LedTransitions Telemetry

To check that blinking stops after turning blinking off, we can check that the `LedTransitions` channel is no longer being emitted.

Add the following assertion after disabling blinking:

```python
    time.sleep(1)  # Wait one second to let any in-progress telemetry be sent
    # Save reference to current telemetry history so we can search against future telemetry
    telem_after_blink_off = fprime_test_api.telemetry_history.size()
    time.sleep(2)  # Wait to receive telemetry after stopping blinking
    # Assert that blinking has stopped and that LedTransitions is no longer updating
    fprime_test_api.assert_telemetry_count(
        0, "LedBlinker.led.LedTransitions", start=telem_after_blink_off
    )
```

Because telemetry is sent once per second, some outdated telemetry may be sent after disabling blinking.
After waiting 1 second for old telemetry to be sent, we can save a reference to telemetry history size, allowing us to search any telemetry received after this point. We then search telemetry using that start reference to assert that 0 new values of the LedTransitions channel are received after this start time. This allows us to confirm that blinking is no longer occurring.


Finally, while blinking is enabled, verify that LedTransitions increments over time.
To verify this, `fprime_test_api.assert_telemetry_count` can be used to wait for and collect a number of LedTransitions values.

```python
    # Assert that the LedTransitions channel increments
    results = fprime_test_api.assert_telemetry_count(
        predicates.greater_than(2), "LedBlinker.led.LedTransitions", timeout=4
    )
    ascending = True
    prev = None
    for res in results:
        if prev is not None:
            if not res.get_val() > prev.get_val():
                ascending = False
                fprime_test_api.log(
                    f"LedBlinker.led.LedTransitions not in ascending order: First ({prev.get_val()}) Second ({res.get_val()})"
                )
        prev = res
    assert fprime_test_api.test_assert(
        ascending, "Expected all LedBlinker.led.LedTransitions updates to ascend.", True
    )
```

### Next Step: [Conclusion](./conclusion.md).
