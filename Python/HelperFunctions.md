### Parsing an ARN and breaking it down into components
The following is a function that will parse an Amazon Resource Name (ARN) according to the [documentation](http://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html).  It was contributed by Ashish Gore, goreashi@amazon.com.

#### Example
```
print(parse_arn("arn:aws:elasticloadbalancing:us-east-1:217082351882:loadbalancer/app/ForAPI2/4a3ae722033cb007"))
```

#### Returns
```
{'account': '217082351882', 'resource': 'app/ForAPI2/4a3ae722033cb007', 'service': 'elasticloadbalancing', 'resourcetype': 'loadbalancer', 'region': 'us-east-1', 'partition': 'aws', 'arn': 'arn'}
```

```python
def parse_arn(arn):
	elements = arn.split(':')
	result = {'arn': elements[0],
			'partition': elements[1],
			'service': elements[2],
			'region': elements[3],
			'account': elements[4]
		}
 
	if len(elements) > 5:
		tokens = elements[5].split("/")
 
		if len(tokens) > 1:
			result['resourcetype'] = tokens[0]
			result['resource'] = "/".join(tokens[1:])
		else:
			if len(elements) > 6:
				result["resourcetype"] = elements[5]
				result['resource'] = ".".join(elements[6:])  
			else:
				result['resourcetype'] = ""
				result['resource'] = elements[5]    
	return result
```