# ADC Publisher (`adc_publisher`, package `uuv_adc_driver`, branch `feature/sensor-adc`)

**Status:** branch. Used by the operator, not by autonomy.

**Subscription(s):** none. It reads the Navigator ADC.

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /adc/data | *std_msgs/msg/Float32MultiArray* | – | – | Every Navigator ADC channel |

### Functionality:

Reads all ADC channels and publishes them. Its launch file uses a namespace that must be removed (see [Integration fixes](../integration_fixes.md)).
