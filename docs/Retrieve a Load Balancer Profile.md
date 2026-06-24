# Retrieve a Load Balancer Profile  

This request retrieves a load balancer profile specified by the name in the URL.  

## **Request**  

### **GET** `/load_balancer/profiles/{name}`  

### **Auditing**  
Calling this method generates the following auditing event:  
`is.load-balancer.load-balancer-profile.read`  

### **Path Parameters**  
| Parameter | Required | Type   | Description |
|-----------|----------|--------|-------------|
| `name`    | Yes      | string | The load balancer profile name. |

**Possible values:**  
- Length: `1 ≤ length ≤ 63`  
- Regular expression: `^([a-z]|[a-z][-a-z0-9]*[a-z0-9]|[0-9][-a-z0-9]*([a-z]|[-a-z][-a-z0-9]*[a-z0-9]))$`  
- **Example:** `network-fixed`  

### **Query Parameters**  
| Parameter   | Required | Type   | Description |
|-------------|----------|--------|-------------|
| `version`   | Yes      | string | The API version, in format `YYYY-MM-DD`. For the API behavior documented here, specify any date between `2024-11-19` and `2025-03-18`. |
| `generation` | Yes      | int32  | The infrastructure generation. For the API behavior documented here, specify `2`. |

**Possible values:**  
- `version`: Regular expression: `^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$`  
  - **Example:** `2024-06-23`  
- `generation`: Allowable values: `[2]`  

---

## **Example Request**  

```sh
curl -X GET "$vpc_api_endpoint/v1/load_balancer/profiles/$profile_name?version=2025-03-18&generation=2" \
     -H "Authorization: Bearer $iam_token"
