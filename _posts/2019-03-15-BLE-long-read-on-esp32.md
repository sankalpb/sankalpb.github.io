---
title: How to enable BLE long read on ESP32
category: [bluetooth-lowenergy, embedded]
---
- If you have tried sending more than 22 bytes using the ``esp_ble_gatts_send_response`` API, you may have encountered the warning.

```
BT_GATT: attribute value too long, to be truncated to 22
```

- The generic attribute protocol only supports packet sizes of 23 by default. 
- Excluding ATT protocol headers this leaves 22 bytes of payload.
- In order to send bigger payloads, your application needs to support the long read command which sends your payload in multiple packets.
- Within the ESP32 bluedroid framework, both read and read long commands trigger the ``ESP_GATTS_READ_EVT`` event.
- Read long commands can be differentiated using the ``is_long`` and ``offset`` fields in read command params.

{% highlight C %}
    struct gatts_read_evt_param {
        uint16_t conn_id;               /*!< Connection id */
        uint32_t trans_id;              /*!< Transfer id */
        esp_bd_addr_t bda;              /*!< The bluetooth device address which been read */
        uint16_t handle;                /*!< The attribute handle */
        uint16_t offset;                /*!< Offset of the value, if the value is too long */
        bool is_long;                   /*!< The value is too long or not */
        bool need_rsp;                  /*!< The read operation need to do response */
    } read;
{% endhighlight %}

- For a long read, the ``ESP_GATTS_READ_EVT`` is fired multiple times with increasing offsets. 
- So to tie everything together, use the ``offset`` parameter to locate the correct segment within the send buffer. When buffer has been read to entirety send an empty segment.
{% highlight C %}
typedef struct {
    uint8_t               *buf;
    int                   len;
} partial_buf;

static void gatts_profile_event_handler(esp_gatts_cb_event_t event, esp_gatt_if_t gatts_if, esp_ble_gatts_cb_param_t *param) {

   // Other events

   case ESP_GATTS_READ_EVT: {
   	/* struct partial_buf buf contains data to transmit */
        if (param->read.is_long && buf.buf) {
           rsp.attr_value.handle = param->read.handle;
	   int remaining_read = (buf.len - param->read.offset);
	   rsp.attr_value.len = (remaining_read) > 22 ? 22: remaining_read;
	   memcpy(rsp.attr_value.value, buf.buf + param->read.offset, rsp.attr_value.len);
	   if (offset + rsp.attr_value.len >= buf.len) {
	      if (buf.buf) {
	      	 free(buf.buf);
		 buf.buf = NULL;
	      }
	      buf.len = 0;
	   }
	}
	esp_ble_gatts_send_response(gatts_if, param->read.conn_id, param->read.trans_id,
                                    ESP_GATT_OK, &rsp);
        break;
   }
   // Other events
}
{% endhighlight %}
