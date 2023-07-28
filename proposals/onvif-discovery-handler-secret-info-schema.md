# Onvif Discovery Handler Secret data schema

This document discusses the motivation and propose a schema for Onvif discovery handler to manage secret data passed by Akri Agent.

Currently, our sample Onvif discovery handler does not support discovering Onvif cameras that require authentication. This makes the discovery handler less useful as in most productive systems, authentication has to be enabled.  Without the capability to discover cameras that require 
authentication, Akri Onvif cameras cannot be used.

There are two attributes required for Onvif discovery handler to perform authenticated discovery:
1. an id that can unique identify a camera
2. a credential (username/password) to authenticate the access to a camera

Recently, a new field called [`discoveryProperties`](./secret-management-for-discovery-handler.md) was added in Akri Configuration, information
about credentials is specified in `discoveryProperties` and Akri Agent will read and pass those data to discovery handler.

One possible approach to enable authenticated discovery in Onvif discovery handler is to leverage the `discoverProperties`. We can put information about credentials for discovered cameras in `discoverProperties`, when Onvif discovery handler discovers a camera, it uses the camera id to look up the credential from the credentials information in `discoverProperties` and uses the credential to access the camera.

The following sections describe what is used to unique identify an Onvif camera and the schema used for organizing credential information in Akri Configuration `discoverProperties`.

# Identify Onvif cameras for authentication
We use the address property of the Endpoint Reference [ONVIF Core Specification 7.3.1 Endpoint reference] as the device id to look up
username/password credentials.  The address property in Endpoint Reference is in the Uniform Resource Name: Universally Unique Identifier (URN:UUID) format.
The same UUID can be retrieved by the ```GetEndpointReference``` command after a camera is discovered by Probe message.  Below is an example of ProbeMatch message, the address property in the Endpoint Reference is highlighted.  In this example, we use the uuid (`3fa1fe68-b915-4053-a3e1-ac15a21f5f91`) as the device id string for credential lookup.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<SOAP-ENV:Envelope …>
    <SOAP-ENV:Header>
        <wsa:MessageID>…</wsa:MessageID>
        <wsa:RelatesTo>…</wsa:RelatesTo>
        <wsa:To >…</wsa:To>
        <wsa:Action >…</wsa:Action>
    </SOAP-ENV:Header>
    <SOAP-ENV:Body>
        <wsdd:ProbeMatches>
            <wsdd:ProbeMatch>
                <wsa:EndpointReference>
                     <wsa:Address>urn:uuid:3fa1fe68-b915-4053-a3e1-ac15a21f5f91</wsa:Address>
                     <wsa:ReferenceProperties></wsa:ReferenceProperties>
                     <wsa:PortType>ttl</wsa:PortType>
                </wsa:EndpointReference>
                <wsdd:Types>tdn:NetworkVideoTransmitter</wsdd:Types>
                <wsdd:Scopes>…</wsdd:Scopes>
                <wsdd:XAddrs>…</wsdd:XAddrs>
                <wsdd:MetadataVersion>…</wsdd:MetadataVersion>
             </wsdd:ProbeMatch>
          </wsdd:ProbeMatches>
        </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

# Organize Credentials in Akri Configuration and Kubernetes Secrets
All secret information are kept in Kubernetes Secrets, in Akri Configuration. We need to create a mapping for the secret information so Agent
can read the secret information and pass it with the mapping to Onvif Discovery Handler.  With the mapping and secret information, Onvif
Discovery Handler can perform credential look up using device ids described in previous section.

There are 3 ways to organize secret information:
1.	Device credential list
2.	Device credential ref list
3.	Device credential entry

All three ways can be used in the same Akri Configuration, the order above is the order of Onvif Discovery Handler processing the secret 
nformation. If there is any secret information duplication between different groups, the latter overwrites the prior entries.
If there is any duplication within the same group, it’s up to the Onvif Discovery Handler to decide which one wins when processing
the entries, and it’s not guaranteed the order is always the same.

## Device credential list
Onvif Discovery Handler first looks for a key named “`device_credential_list`” and expects its value points to a string array of credential
lists in json format.  The value is an array of credential list which points to a Kubernetes secret, the secret is expected in json format.

Here is an example of Device credential list.
In Akri Configuration, an entry named “`device_credential_list`” is listed in discoveryProperties.  The value contains an array of device secret lists.  The device secret lists are entries that point to the actual Kubernetes Secret key.

```yaml
    discoveryProperties:
    - name: "device_credential_list"
      value: |+
        [
          "secret_list1",
          "secret_list2"
        ]
    - name: "secret_list1"
      valueFrom:
        secretKeyRef:
          name: "onvif-auth-secret"
          namespace: "onvif-auth-secret-namespace"
          key: "credential_list1"
          optional: false
    - name: "secret_list2"
      valueFrom:
        secretKeyRef:
          name: "onvif-auth-secret"
          namespace: "onvif-auth-secret-namespace"
          key: "credential_list2"
          optional: false
```

In Kubernetes Secret `onvif-auth-secret`, the `credential_list1` and `credential_list2` contain the actual secret information for a list of devices.  The entry uses the device id as key and the value is a json object with username and password.  The password can be optionally encoded with base64 (with “`base64encoded`” set to true).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: onvif-auth-secret
  namespace: onvif-auth-secret-namespace
type: Opaque
stringData:
  credential_list1: |+
    {
      "6821dc67-8438-5588-1547-4d1349048438" : { "username" : "admin", "password" : "adminpassword" },
      "6a67158b-42b1-400b-8afe-1bec9a5d7919" : { "username" : "user1", "password" : "SGFwcHlEYXk=", "base64encoded": true }
    }
  credential_list2: |+
    {
      "5f5a69c2-e0ae-504f-829b-00fcdab169cc" : { "username" : "admin", "password" : "admin" }
    }
```

## Device credential ref list

Device credential ref list is similar to Device credential list except the device ids are listed and the credentials are references 
to another entries in the Akri `discoveryProperties`. The key name for device credential ref list is “`device_credential_ref_list`”.

For example, the device credential ref list below contains an array of “device id”->”credential reference” objects.  The credential of device id “5f5a69c2-e0ae-504f-829b-00fcdab169cc” is referred to (username-> device1_username, password->device1_password).  The device1_username and device1_password are entries in Akri discoverProperties that point to the actual secret information in Kubernetes Secrets.  Note different device ids may use the same secret reference.

```yaml
    - name: "device_credential_ref_list"
      value: |+
        [
          "secret_ref_list1",
          "secret_ref_list2"
        ]
    - name: "secret_ref_list1"
      value: |+
        {
          "5f5a69c2-e0ae-504f-829b-00fcdab169cc" : { "username_ref" : "device1_username", "password_ref" : "device1_password" },
          "6a67158b-42b1-400b-8afe-1bec9a5d7909":  { "username_ref" : "device2_username", "password_ref" : "device2_password" }
        }
    - name: "secret_ref_list2"
      value: |+
        {
          "7a67158b-42b1-400b-8afe-1bec9a5d790a":  { "username_ref" : "device2_username", "password_ref" : "device2_password" }
        }
    - name: "device1_username"
      valueFrom:
        secretKeyRef:
          name: "onvif-auth-secret"
          namespace: "onvif-auth-secret-namespace"
          key: "camera1_username"
          optional: false
    - name: "device1_password"
      valueFrom:
        secretKeyRef:
          name: "onvif-auth-secret"
          namespace: "onvif-auth-secret-namespace"
          key: "camera1_password"
          optional: true
    - name: "device2_username"
      valueFrom:
        secretKeyRef:
          name: "onvif-auth-secret"
          namespace: "onvif-auth-secret-namespace"
          key: "camera2_username"
          optional: false
    - name: "device2_password"
      valueFrom:
        secretKeyRef:
          name: "onvif-auth-secret"
          namespace: "onvif-auth-secret-namespace"
          key: "camera2_password"
          optional: true
```

The actual secret information is in Kubernetes Secret `onvif-auth-secret`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: onvif-auth-secret
  namespace: onvif-auth-secret-namespace
type: Opaque
stringData:
  camera1_username: "admin"
  camera1_password: "admin"
  camera2_username: "cam2_user"
  camera2_password: "cam2_pwd"
```

## Device credential entry
Device credential entry is a direct mapping from device id to its credential, using "`username_<device-id>`" and "`password_<device id>`" as key names.

```yaml
    discoveryProperties:
    - name: "username_6a67158b-42b1-400b-8afe-1bec9a5d7909"
      valueFrom:
        secretKeyRef:
          name: "onvif-auth-secret"
          namespace: "onvif-auth-secret-namespace"
          key: "camera1_username"
          optional: false
    - name: "password_6a67158b-42b1-400b-8afe-1bec9a5d7909"
      valueFrom:
        secretKeyRef:
          name: "onvif-auth-secret"
          namespace: "onvif-auth-secret-namespace"
          key: "camera1_password"
          optional: false
```

The actual secret information is in Kubernetes Secret `onvif-auth-secret`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: onvif-auth-secret
  namespace: onvif-auth-secret-namespace
type: Opaque
stringData:
  camera1_username: "admin"
  camera1_password: "admin"
```
