## Delete a Load Balancer Listener

This request deletes a load balancer listener. This operation cannot be reversed. For this operation to succeed, the listener must not be the target of another load balancer listener.

### **DELETE** `/load_balancers/{load_balancer_id}/listeners/{id}`

### **Authorization**
To call this method, you must be assigned one or more IAM access roles that include the following action. You can check your access by going to Users > User > Access.

`is.load-balancer.load-balancer.manage`

### **Auditing**
Calling this method generates the following auditing event.

`is.load-balancer.load-balancer-listener.delete`

### **Request**

#### **Path Parameters**

| Parameter          | Required | Type   | Description                  | Constraints |
|-------------------|----------|--------|------------------------------|-------------|
| `load_balancer_id` | **Yes**  | string | The load balancer identifier | 1 ≤ length ≤ 64, must match `^[-0-9a-z_]+$` |
| `id`              | **Yes**  | string | The listener identifier       | 1 ≤ length ≤ 64, must match `^[-0-9a-z_]+$` |

#### **Query Parameters**

| Parameter  | Required | Type   | Description                                      | Constraints |
|------------|----------|--------|--------------------------------------------------|-------------|
| `version`  | **Yes**  | string | The API version in `YYYY-MM-DD` format         | Length = 10, must match `^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$` |
| `generation` | **Yes**  | int32  | The infrastructure generation (must be `2`) | Allowable value: `2` |

#### **Example Request**
```sh
curl -X DELETE "$vpc_api_endpoint/v1/load_balancers/$load_balancer_id/listeners/$listener_id?version=2025-03-18&generation=2" \
-H "Authorization: Bearer $iam_token"
```

### **Response**

| Status Code | Description |
|------------|-------------|
| `202`      | The listener deletion request was accepted. |
| `404`      | A load balancer or listener with the specified identifier could not be found. |
| `409`      | The listener is in use and cannot be deleted. |
| `403`      | The user does not have permission to delete the listener. |

---

## Retrieve a Load Balancer Listener

This request retrieves a single listener specified by the identifier in the URL path.

### **GET** `/load_balancers/{load_balancer_id}/listeners/{id}`

### **Authorization**
To call this method, you must be assigned one or more IAM access roles that include the following action. You can check your access by going to Users > User > Access.

`is.load-balancer.load-balancer.view`

### **Auditing**
Calling this method generates the following auditing event.

`is.load-balancer.load-balancer-listener.read`

### **Request**

#### **Path Parameters**

| Parameter          | Required | Type   | Description                  | Constraints |
|-------------------|----------|--------|------------------------------|-------------|
| `load_balancer_id` | **Yes**  | string | The load balancer identifier | 1 ≤ length ≤ 64, must match `^[-0-9a-z_]+$` |
| `id`              | **Yes**  | string | The listener identifier       | 1 ≤ length ≤ 64, must match `^[-0-9a-z_]+$` |

#### **Query Parameters**

| Parameter  | Required | Type   | Description                                      | Constraints |
|------------|----------|--------|--------------------------------------------------|-------------|
| `version`  | **Yes**  | string | The API version in `YYYY-MM-DD` format         | Length = 10, must match `^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$` |
| `generation` | **Yes**  | int32  | The infrastructure generation (must be `2`) | Allowable value: `2` |

#### **Example Request**
```sh
curl -X GET "$vpc_api_endpoint/v1/load_balancers/$load_balancer_id/listeners/$listener_id?version=2025-03-18&generation=2" \
-H "Authorization: Bearer $iam_token"
```

### **Response**

#### **Response Body**

| Property                | Required | Type    | Description |
|-------------------------|----------|---------|-------------|
| `accept_proxy_protocol` | **Yes**  | boolean | If `true`, this listener accepts and forwards PROXY protocol information. |
| `created_at`            | **Yes**  | date-time | The creation timestamp. |
| `href`                  | **Yes**  | string  | The URL for this load balancer listener. |
| `id`                    | **Yes**  | string  | The unique identifier for this load balancer listener. |
| `port`                  | **Yes**  | integer | The listener port number or the inclusive lower bound of the port range. |
| `port_max`              | **Yes**  | integer | The upper bound of the range of ports used by this listener. |
| `port_min`              | **Yes**  | integer | The lower bound of the range of ports used by this listener. |
| `protocol`              | **Yes**  | string  | The listener protocol. Possible values: `http`, `https`, `tcp`, `udp`. |
| `provisioning_status`   | **Yes**  | string  | The provisioning status. Possible values: `active`, `create_pending`, `delete_pending`, `failed`, `update_pending`. |
| `certificate_instance`  | No       | object  | The certificate instance used for SSL termination. |
| `connection_limit`      | No       | integer | The concurrent connection limit. Range: `1` to `15000`. |
| `default_pool`          | No       | object  | The default pool for this listener. |
| `https_redirect`        | No       | object  | The target listener for HTTPS redirects. |
| `idle_connection_timeout` | No    | integer | The idle connection timeout in seconds. Range: `50` to `7200`. |
| `policies`              | No       | array   | The policies for this listener. |

### **Status Codes**

| Status Code | Description |
|------------|-------------|
| `200`      | The listener was retrieved successfully. |
| `404`      | A load balancer with the specified identifier could not be found. |
| `403`      | The user does not have permission to retrieve the listener. |
