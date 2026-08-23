# Invoking another service on the same Thing or on another entity from a service

This document summarizes how, inside ThingWorx, a service implementation calls **another service on itself** or **on another entity**.

**ThingWorx source root (repo convention):** `C:\Users\dxu\Documents\bitbucket\tw-server`  
(Same as **./AGENT-CONTEXT.md §1.1**; paths below are relative to that root.)

---

## 1. Calling another service on the same Thing

Inside one Thing/Entity’s service implementation, to call **another service on this entity**, use `processServiceRequestDirect(serviceName, params)` or `processAPIServiceRequest(serviceName, params)`.

### Example 1: GenericConnector (intra-Thing calls)

`thingworx-platform-common/src/com/thingworx/connectors/GenericConnector.java`

```java
// In a private method, call this Thing's GetEndpointDefinition
ValueCollection endpointDefParams = new ValueCollection(1);
endpointDefParams.SetStringValue("endpointId", endpointDefinitionId);
InfoTable defResultIT = this.processServiceRequestDirect("GetEndpointDefinition", endpointDefParams);

// In GenerateServices, call GetEndpointList and GenerateServiceFromEndpoint on this Thing
ValueCollection params = new ValueCollection();
InfoTable resultInfoTable = this.processServiceRequestDirect("GetEndpointList", params);
// ...
this.processServiceRequestDirect("GenerateServiceFromEndpoint", params);
```

### Example 2: CacheThing (intra-instance call so JS overrides apply)

`thingworx-platform-common/src/com/thingworx/things/cache/custom/CacheThing.java`

```java
// processServiceRequestDirect calls this Thing's LoadEntry and honors JS overrides
InfoTable loaderResult = cacheThingInstance.processServiceRequestDirect("LoadEntry", loadParams);
```

**Takeaway**: When the current execution context is this Thing, use `this.processServiceRequestDirect(serviceName, params)`; the return value is `InfoTable`.

---

## 2. Calling a service on another entity

### 2.1 Target is a local Thing (same platform process)

Resolve the entity via **ThingManager** or **EntityUtilities**, then call `processAPIServiceRequest` or `processServiceRequestDirect` on `IServiceProvider`.

#### Example: Log calls the “log retrieval strategy” Thing

`thingworx-platform-common/src/com/thingworx/logging/Log.java`

```java
String logRetrievalStrategyThingName = (String) LoggingSubsystem.getSubsystemInstance().getConfigurationSetting(...);
Thing logRetrievalStrategyThing = ThingManager.getInstance().getEntity(logRetrievalStrategyThingName);

if (logRetrievalStrategyThing == null) {
    throw new InvalidRequestException("Log Retrieval Strategy Thing: " + logRetrievalStrategyThingName + " ...");
}

ValueCollection queryParameters = new ValueCollection();
// ... populate parameters
InfoTable results = logRetrievalStrategyThing.processServiceRequestDirect(
    LoggingConstants.RETRIEVE_LOGS_STRATEGY_SERVICE_NAME, queryParameters);
```

#### Example: IndustrialThingShape calls Gateway Thing (local IndustrialGateway)

`thingworx-platform-common/src/com/thingworx/things/connected/IndustrialThingShape.java`

```java
String gatewayName = me.getProperty(INDUSTRIAL_THING).getValue().getStringValue();
Thing thingEntity = ThingManager.getInstance().getEntity(gatewayName);
if (thingEntity instanceof IndustrialGateway) {
    IndustrialGateway gatewayThing = (IndustrialGateway) thingEntity;
    if (gatewayThing.isConnected()) {
        ValueCollection params = new ValueCollection();
        params.put("fullyQualifiedTagAddress", aspects.getAspect(Aspects.ASPECT_TAGADDRESS));
        return gatewayThing.callService("GetDiagnosticDigest", params, BaseTypes.NOTHING);
    }
}
```

Note: Here the target is **RemoteThing/IndustrialGateway**, so `callService` is used for remote invocation; for a purely local Thing, use `processServiceRequestDirect` / `processAPIServiceRequest`.

---

### 2.2 Target is any local entity (Thing / Resource / Subsystem, etc.)

Resolve entity by name and type, then invoke as `IServiceProvider`.

#### Example: ExternalApplication calls an arbitrary entity by type and name

`thingworx-platform-common/src/com/thingworx/things/connected/ExternalApplication.java`

```java
final RootEntity entity = EntityUtilities.findEntity(entityName,
    RelationshipTypes.ThingworxRelationshipTypes.valueOf(entityType));
if (entity == null || !(entity instanceof IServiceProvider)) {
    throw new InvalidRequestException("Entity not found or invalid", ...);
}
final ValueCollection serviceParams = (parameters == null || parameters.getRowCount() == 0)
    ? new ValueCollection() : parameters.getFirstRow();
return ((IServiceProvider) entity).processAPIServiceRequest(serviceName, serviceParams);
```

---

### 2.3 Target is RemoteThing (edge / remotely connected Thing)

For **RemoteThing** (and subclasses such as IndustrialGateway) you must use `callService`, not `processServiceRequestDirect` (which runs locally).

#### Example: AlertProcessingSubsystem calls RemoteThing

`thingworx-platform-common/src/com/thingworx/system/subsystems/alerts/AlertProcessingSubsystem.java`

```java
final RemoteThing remoteThing = (RemoteThing) things.get(0).getReferenceDirect();
// ...
final InfoTable result = remoteThing.callService("IsAnalyticsServerRunning", null, BaseTypes.BOOLEAN);
```

#### Example: IndustrialThingShape calls Gateway AddIndustrialThing / RemoveIndustrialThing

`thingworx-platform-common/src/com/thingworx/things/connected/IndustrialThingShape.java`

```java
IndustrialGateway gatewayThing = (IndustrialGateway) thingEntity;
ValueCollection params = new ValueCollection();
params.put("thingName", new StringPrimitive(meThingName));
gatewayThing.callServiceAsync("AddIndustrialThing", params, BaseTypes.NOTHING, NOTIFICATION_DELAY);
// or synchronous
gatewayThing.callService("RemoveIndustrialThing", params, BaseTypes.NOTHING);
```

---

### 2.4 Generic helper: MCPCapabilityManager.invokeService

For any `IServiceProvider` you already hold, if you only need “invoke by name and get result”, follow the MCP wrapper pattern (still uses `processAPIServiceRequest` internally):

`thingworx-platform-common/src/com/thingworx/mcp/MCPCapabilityManager.java`

```java
public static JSONObject invokeService(IServiceProvider serviceProvider, String serviceName, ValueCollection valueCollection)
        throws Exception {
    InfoTable result = serviceProvider.processAPIServiceRequest(serviceName, valueCollection);
    // ... convert result to JSONObject
    return jsonResult;
}
```

---

## 3. Obtaining the “current” entity (self reference inside a service)

When a service implementation needs the **currently running entity** (e.g. to pass elsewhere or call self), use:

```java
Thing me = (Thing) ThreadLocalContext.getMeContext();
// or
IServiceProvider myServiceProvider = (IServiceProvider) ThreadLocalContext.getMeContext();
```

To call another service on self, you can also use `this.processServiceRequestDirect(...)` when `this` is the executing Thing.

---

## 4. Summary

| Scenario | Approach | Typical locations |
|----------|----------|-------------------|
| Call **another service on self** | `this.processServiceRequestDirect(serviceName, params)` | GenericConnector, CacheThing |
| Call a **local Thing**’s service | `ThingManager.getInstance().getEntity(name)` then `.processServiceRequestDirect(serviceName, params)` | Log |
| Call **any local entity**’s service | `EntityUtilities.findEntity(name, type)` then `((IServiceProvider) entity).processAPIServiceRequest(serviceName, params)` | ExternalApplication |
| Call a **RemoteThing**’s service | On `RemoteThing`: `remoteThing.callService(serviceName, params, resultType)` or `callServiceAsync(...)` | IndustrialThingShape, AlertProcessingSubsystem |
| Current entity reference | `ThreadLocalContext.getMeContext()` | IndustrialThingShape, StatisticalUtils, etc. |

- **processAPIServiceRequest / processServiceRequestDirect**: for **local** entity service calls in-platform; perform permission and parameter validation.  
- **callService / callServiceAsync**: **RemoteThing** only (connected Things); remote/edge invocation path.
