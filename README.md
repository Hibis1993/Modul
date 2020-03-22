/ *
Copyright (c) 2009-2019 Роджер Лайт <roger@atchoo.org>

Все права защищены. Эта программа и сопутствующие материалы
предоставляются в соответствии с условиями Общественной лицензии Eclipse v1.0
и Eclipse Distribution License v1.0, которые сопровождают это распространение.

Общественная лицензия Eclipse доступна по адресу
   http://www.eclipse.org/legal/epl-v10.html
и Eclipse Distribution License доступна по адресу
  http://www.eclipse.org/org/documents/edl-v10.php.

Авторы:
   Roger Light - первоначальная реализация и документация.
* /

#include "config.h"

#include <assert.h>

#include "mosquitto.h"
#include "logging_mosq.h"
#include "memory_mosq.h"
#include "messages_mosq.h"
#include "mqtt_protocol.h"
#include "net_mosq.h"
#include "packet_mosq.h"
#include "property_mosq.h"
#include "read_handle.h"

static void connack_callback (свойства struct mosquitto * mosq, uint8_t reason_code, uint8_t connect_flags, свойства const mosquitto_property *)
{
	log__printf (mosq, MOSQ_LOG_DEBUG, "Клиент% s получил CONNACK (% d)", mosq-> id, код_ причины);
	if (reason_code == MQTT_RC_SUCCESS) {
		mosq- >connects = 0;
	}
	pthread_mutex_lock (& ​​mosq-> callback_mutex);
	если (mosq-> on_connect) {
		mosq-> in_callback = true;
		mosq-> on_connect (mosq, mosq-> userdata, reason_code);
		mosq-> in_callback = false;
	}
	если (mosq-> on_connect_with_flags) {
		mosq-> in_callback = true;
		mosq-> on_connect_with_flags (mosq, mosq-> userdata, reason_code, connect_flags);
		mosq-> in_callback = false;
	}
	если (mosq-> on_connect_v5) {
		mosq-> in_callback = true;
		mosq-> on_connect_v5 (mosq, mosq-> userdata, код_ причины, connect_flags, свойства);
		mosq-> in_callback = false;
	}
	pthread_mutex_unlock (& ​​mosq-> callback_mutex);
}


int handle__connack (struct mosquitto * mosq)
{
	uint8_t connect_flags;
	uint8_t reason_code;
	int rc;
	mosquitto_property * properties = NULL;
	char * clientid = NULL;

	утверждают (Mosq);
	rc = packet__read_byte (& mosq-> in_packet, & connect_flags);
	if (rc) вернуть rc;
	rc = packet__read_byte (& mosq-> in_packet, & reason_code);
	if (rc) вернуть rc;

	if (mosq-> protocol == mosq_p_mqtt5) {
		rc = property__read_all (CMD_CONNACK, & mosq-> in_packet, & properties);

		if (rc == MOSQ_ERR_PROTOCOL && reason_code == CONNACK_REFUSED_PROTOCOL_VERSION) {
			/ * Это может произойти, потому что мы подключаемся к брокеру v3.x и
			 * он ответил «неприемлемой версией протокола», но с
			 * v3 КОННАК. * /

			connack_callback (mosq, MQTT_RC_UNSUPPORTED_PROTOCOL_VERSION, connect_flags, NULL);
			возврат RC;
		} если if (rc) {
			возврат RC;
		}
	}

	mosquitto_property_read_string (свойства, MQTT_PROP_ASSIGNED_CLIENT_IDENTIFIER, & clientid, false);
	если (ClientID) {
		если (mosq-> ID) {
			/ * Нам отправлен идентификатор клиента, но он уже есть. Эта
			 * не должно случиться * /
			бесплатно (ClientID);
			mosquitto_property_free_all (& свойства);
			return MOSQ_ERR_PROTOCOL;
		} Еще {
			mosq-> id = clientid;
			clientid = NULL;
		}
	}

	mosquitto_property_read_byte (свойства, MQTT_PROP_MAXIMUM_QOS, & mosq-> Maximum_qos, false);
	mosquitto_property_read_int16 (свойства, MQTT_PROP_RECEIVE_MAXIMUM, & mosq-> msgs_out.inflight_maximum, false);
	mosquitto_property_read_int16 (свойства, MQTT_PROP_SERVER_KEEP_ALIVE, & mosq-> keepalive, false);
	mosquitto_property_read_int32 (свойства, MQTT_PROP_MAXIMUM_PACKET_SIZE, & mosq-> Maximum_packet_size, false);

	mosq-> msgs_out.inflight_quota = mosq-> msgs_out.inflight_maximum;

	connack_callback (mosq, код_причины, connect_flags, свойства);
	mosquitto_property_free_all (& свойства);

	Переключатель (reason_code) {
		случай 0:
			pthread_mutex_lock (& ​​mosq-> state_mutex);
			if (mosq-> state! = mosq_cs_disconnecting) {
				mosq-> state = mosq_cs_active;
			}
			pthread_mutex_unlock (& ​​mosq-> state_mutex);
			message__retry_check (Mosq);
			вернуть MOSQ_ERR_SUCCESS;
		Дело 1:
		случай 2:
		случай 3:
		дело 4:
		дело 5:
			вернуть MOSQ_ERR_CONN_REFUSED;
		дефолт:
			return MOSQ_ERR_PROTOCOL;
	}
}
