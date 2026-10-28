# Apache Struts Patterns

This document defines comprehensive detection patterns for Apache Struts 1.x and 2.x applications. These patterns help identify all aspects of Struts-based web applications including configuration, actions, forms, validation, interceptors, and view technologies for modernization to Spring Boot.

## Table of Contents

- [Version Detection](#version-detection)
- [Struts 1.x Patterns](#struts-1x-patterns)
  - [Configuration Files](#struts-1x-configuration-files)
  - [Action Classes](#struts-1x-action-classes)
  - [Form Beans](#struts-1x-form-beans)
  - [Validation](#struts-1x-validation)
  - [Tag Libraries](#struts-1x-tag-libraries)
  - [Tiles 1.x Integration](#struts-1x-tiles-integration)
- [Struts 2.x Patterns](#struts-2x-patterns)
  - [Configuration Files](#struts-2x-configuration-files)
  - [Annotation-Based Configuration](#struts-2x-annotation-based-configuration)
  - [Action Classes](#struts-2x-action-classes)
  - [Convention Plugin](#struts-2x-convention-plugin)
  - [Validation Framework](#struts-2x-validation-framework)
  - [Interceptors](#struts-2x-interceptors)
  - [Tag Libraries](#struts-2x-tag-libraries)
  - [Tiles 2.x/3.x Integration](#struts-2x-tiles-integration)
- [Common Patterns](#common-patterns)
- [Migration Summary](#migration-summary)

---

## Version Detection

### Struts 1.x Indicators

Presence of any of these patterns indicates Struts 1.x:

- `struts-config.xml` configuration file
- Java classes extending `org.apache.struts.action.Action`
- Java classes extending `org.apache.struts.action.ActionForm`
- JSP taglib URIs: `/tags/struts-html`, `/tags/struts-bean`, `/tags/struts-logic`
- `tiles-defs.xml` with Struts 1.x DTD

### Struts 2.x Indicators

Presence of any of these patterns indicates Struts 2.x:

- `struts.xml` configuration file
- Java classes extending `com.opensymphony.xwork2.ActionSupport`
- `@Action` annotation from `org.apache.struts2.convention.annotation`
- JSP taglib URI: `/struts-tags`
- `StrutsPrepareAndExecuteFilter` in web.xml
- `tiles.xml` with Tiles 2/3 namespace

### Mixed Version Detection

Some applications may be in the process of migrating from Struts 1.x to 2.x and contain both:

- Both `struts-config.xml` and `struts.xml` present
- Mix of Struts 1.x and 2.x dependencies
- When detected, add `"Struts Version Migration In Progress"` to `ModernizationRequirements.migration_considerations`

---

## Struts 1.x Patterns

### Struts 1.x Configuration Files

#### struts-config.xml

|| Pattern | `struts-config\.xml` (filename pattern) |
||---------|----------------------------------------|
|| **File Types** | `struts-config.xml` |
|| **Indicates** | Struts 1.x application configuration |
|| **Schema Entity** | `ModernizationRequirements.source_technology = STRUTS_1` |

**Example:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE struts-config PUBLIC
    "-//Apache Software Foundation//DTD Struts Configuration 1.3//EN"
    "http://struts.apache.org/dtds/struts-config_1_3.dtd">
<struts-config>
    <form-beans>
        <form-bean name="orderForm" type="com.example.forms.OrderForm"/>
    </form-beans>
    
    <global-forwards>
        <forward name="welcome" path="/welcome.do"/>
        <forward name="logout" path="/logout.do" redirect="true"/>
    </global-forwards>
    
    <action-mappings>
        <action path="/orders/create"
                type="com.example.actions.CreateOrderAction"
                name="orderForm"
                scope="request"
                validate="true"
                input="/WEB-INF/pages/orders/form.jsp">
            <forward name="success" path="/orders/list.do" redirect="true"/>
            <forward name="error" path="/WEB-INF/pages/orders/form.jsp"/>
        </action>
    </action-mappings>
    
    <controller processorClass="org.apache.struts.tiles.TilesRequestProcessor"/>
    
    <message-resources parameter="ApplicationResources"/>
    
    <plug-in className="org.apache.struts.tiles.TilesPlugin">
        <set-property property="definitions-config" value="/WEB-INF/tiles-defs.xml"/>
    </plug-in>
</struts-config>
```

**Extraction Notes:**
- Parse `<form-beans>` → each `<form-bean>` becomes a `FormDefinition` with `model_object` = form bean class
- Parse `<action-mappings>` → each `<action>` becomes:
  - `ServiceEndpoint` with `path` from action path, method typically POST or GET
  - Extract `name` attribute to link to form bean
  - Extract `scope` → if "session", note session state management
  - Extract `validate="true"` → form has validation
  - Extract `input` → JSP path for form display
  - Extract `<forward>` elements → `NavigationFlow` entries with `from_view` = action path, `to_view` = forward path
- Parse `<global-forwards>` → shared navigation rules
- Parse `<controller>` → identify custom request processor (e.g., TilesRequestProcessor)
- Parse `<message-resources>` → i18n resource bundle location
- Parse `<plug-in>` → identify plugins (Tiles, Validator, etc.)

**Migration Path:**
- Actions → Spring `@Controller` methods with `@RequestMapping`
- Form beans → Spring form-backing objects with validation annotations
- Forwards → Spring redirects or view returns
- Message resources → Spring `MessageSource` configuration

---

#### Form Beans Section

|| Pattern | `<form-beans>` |
||---------|----------------|
|| **File Types** | `struts-config.xml` |
|| **Indicates** | ActionForm definitions |
|| **Schema Entity** | `FormDefinition` |

**Example:**
```xml
<form-beans>
    <form-bean name="loginForm" type="com.example.forms.LoginForm"/>
    <form-bean name="orderForm" type="com.example.forms.OrderForm"/>
    <form-bean name="dynamicForm" type="org.apache.struts.action.DynaActionForm">
        <form-property name="username" type="java.lang.String"/>
        <form-property name="password" type="java.lang.String"/>
    </form-bean>
</form-beans>
```

**Extraction Notes:**
- Each `<form-bean>` → `FormDefinition.name` and `model_object`
- For `DynaActionForm`, extract `<form-property>` elements → `FormField` entries
- Link form beans to actions via the `name` attribute in `<action>` mappings

---

#### Action Mappings Section

|| Pattern | `<action-mappings>` |
||---------|---------------------|
|| **File Types** | `struts-config.xml` |
|| **Indicates** | URL-to-Action mappings |
|| **Schema Entity** | `ServiceEndpoint`, `NavigationFlow` |

**Example:**
```xml
<action-mappings>
    <action path="/login"
            type="com.example.actions.LoginAction"
            name="loginForm"
            scope="request"
            validate="true"
            input="/login.jsp">
        <forward name="success" path="/home.do"/>
        <forward name="failure" path="/login.jsp"/>
    </action>
    
    <action path="/orders/view"
            type="com.example.actions.ViewOrderAction"
            parameter="id">
        <forward name="success" path="/WEB-INF/pages/orders/detail.jsp"/>
    </action>
</action-mappings>
```

**Extraction Notes:**
- `path` → `ServiceEndpoint.path` (URL pattern, typically ends in `.do` or `.action`)
- `type` → Java class implementing the action (see Action Classes section)
- `name` → references form bean by name
- `scope` → if "session", note `NavigationFlow.state_management = SESSION` and add `"Session State"` to `migration_considerations`
- `validate` → if true, form has validation requirements
- `input` → JSP path to return on validation failure
- `parameter` → custom parameter for DispatchAction or request parameter
- `<forward>` → `NavigationFlow` with result code and target path/action

---

#### Global Forwards Section

|| Pattern | `<global-forwards>` |
||---------|---------------------|
|| **File Types** | `struts-config.xml` |
|| **Indicates** | Application-wide navigation rules |
|| **Schema Entity** | `NavigationFlow` |

**Example:**
```xml
<global-forwards>
    <forward name="welcome" path="/index.jsp"/>
    <forward name="logout" path="/logout.do" redirect="true"/>
    <forward name="unauthorized" path="/error/unauthorized.jsp"/>
</global-forwards>
```

**Extraction Notes:**
- Each `<forward>` → `NavigationFlow` entry available from any action
- `redirect="true"` → HTTP redirect (client-side) vs. server forward

---

#### Message Resources

|| Pattern | `<message-resources\s+parameter=["'][^"']+["']` |
||---------|------------------------------------------------|
|| **File Types** | `struts-config.xml` |
|| **Indicates** | i18n resource bundle configuration |
|| **Schema Entity** | `ViewRequirement.i18n_support = true` |

**Example:**
```xml
<message-resources parameter="ApplicationResources" null="false"/>
<message-resources parameter="ModuleResources" key="module"/>
```

**Extraction Notes:**
- `parameter` → resource bundle base name (e.g., `ApplicationResources.properties`)
- Multiple message-resources can be configured with different keys
- Maps to Spring `MessageSource` in migration

---

### Struts 1.x Action Classes

#### Action Class

|| Pattern | `extends\s+Action\b` |
||---------|----------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Struts 1.x action implementation |
|| **Schema Entity** | `ServiceEndpoint`, `ServiceOperation` |

**Example:**
```java
import org.apache.struts.action.Action;
import org.apache.struts.action.ActionForm;
import org.apache.struts.action.ActionForward;
import org.apache.struts.action.ActionMapping;

public class CreateOrderAction extends Action {
    
    @Override
    public ActionForward execute(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        
        OrderForm orderForm = (OrderForm) form;
        
        // Validate
        ActionErrors errors = orderForm.validate(mapping, request);
        if (!errors.isEmpty()) {
            saveErrors(request, errors);
            return mapping.getInputForward();
        }
        
        // Business logic
        OrderService orderService = new OrderService();
        Order order = orderService.createOrder(orderForm);
        
        // Store result
        request.setAttribute("order", order);
        
        return mapping.findForward("success");
    }
}
```

**Extraction Notes:**
- Extract class name and package → action identifier
- `execute()` method → primary action logic
- `mapping.findForward("name")` → navigation outcome references forward in struts-config.xml
- `ActionForm` casting → links to form bean
- `request.setAttribute()` / `request.getAttribute()` → view model data
- Extract business logic calls → `ServiceOperation`
- Thread-safety concern: Actions are singletons in Struts 1.x

**Migration Path:**
- Convert to Spring `@Controller` class
- `execute()` method → `@PostMapping` or `@GetMapping` method
- `ActionForm` parameter → `@Valid @ModelAttribute` command object
- `mapping.findForward()` → return view name or `RedirectView`
- Request attributes → `Model.addAttribute()`

---

#### DispatchAction Class

|| Pattern | `extends\s+DispatchAction\b` |
||---------|------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Multi-method Struts 1.x action |
|| **Schema Entity** | `ServiceEndpoint` (multiple endpoints) |

**Example:**
```java
import org.apache.struts.actions.DispatchAction;

public class OrderDispatchAction extends DispatchAction {
    
    public ActionForward list(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response) {
        // List orders
        List<Order> orders = orderService.findAll();
        request.setAttribute("orders", orders);
        return mapping.findForward("list");
    }
    
    public ActionForward view(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response) {
        String id = request.getParameter("id");
        Order order = orderService.findById(Long.parseLong(id));
        request.setAttribute("order", order);
        return mapping.findForward("view");
    }
    
    public ActionForward delete(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response) {
        String id = request.getParameter("id");
        orderService.delete(Long.parseLong(id));
        return mapping.findForward("success");
    }
}
```

**Extraction Notes:**
- Each public method returning `ActionForward` → separate `ServiceEndpoint`
- Method selection via request parameter (typically `?method=list`)
- Extract each method as independent endpoint
- Configure in struts-config.xml with `parameter` attribute specifying the request parameter name

**struts-config.xml configuration:**
```xml
<action path="/orders"
        type="com.example.actions.OrderDispatchAction"
        parameter="method">
    <forward name="list" path="/WEB-INF/pages/orders/list.jsp"/>
    <forward name="view" path="/WEB-INF/pages/orders/detail.jsp"/>
    <forward name="success" path="/orders.do?method=list" redirect="true"/>
</action>
```

**Migration Path:**
- Split into multiple `@GetMapping` / `@PostMapping` methods in single controller
- Method parameter becomes part of URL path or distinct endpoints

---

#### LookupDispatchAction Class

|| Pattern | `extends\s+LookupDispatchAction\b` |
||---------|-------------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | i18n-aware multi-method action |
|| **Schema Entity** | `ServiceEndpoint` |

**Example:**
```java
import org.apache.struts.actions.LookupDispatchAction;

public class OrderLookupAction extends LookupDispatchAction {
    
    protected Map getKeyMethodMap() {
        Map<String, String> map = new HashMap<>();
        map.put("button.save", "save");
        map.put("button.cancel", "cancel");
        return map;
    }
    
    public ActionForward save(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response) {
        // Save logic
        return mapping.findForward("success");
    }
    
    public ActionForward cancel(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response) {
        return mapping.findForward("cancel");
    }
}
```

**Extraction Notes:**
- `getKeyMethodMap()` maps i18n keys to method names
- Button labels are internationalized
- Similar extraction to DispatchAction

---

### Struts 1.x Form Beans

#### ActionForm Class

|| Pattern | `extends\s+ActionForm\b` |
||---------|--------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Form backing object |
|| **Schema Entity** | `FormDefinition`, `DataStructure` |

**Example:**
```java
import org.apache.struts.action.ActionForm;
import org.apache.struts.action.ActionErrors;
import org.apache.struts.action.ActionMessage;
import org.apache.struts.action.ActionMapping;

public class OrderForm extends ActionForm {
    
    private String customerName;
    private String email;
    private Integer quantity;
    private String productId;
    
    // Getters and setters
    public String getCustomerName() { return customerName; }
    public void setCustomerName(String customerName) { this.customerName = customerName; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public Integer getQuantity() { return quantity; }
    public void setQuantity(Integer quantity) { this.quantity = quantity; }
    
    public String getProductId() { return productId; }
    public void setProductId(String productId) { this.productId = productId; }
    
    @Override
    public ActionErrors validate(ActionMapping mapping, HttpServletRequest request) {
        ActionErrors errors = new ActionErrors();
        
        if (customerName == null || customerName.trim().isEmpty()) {
            errors.add("customerName", new ActionMessage("error.name.required"));
        }
        
        if (email == null || !email.matches("^[A-Za-z0-9+_.-]+@(.+)$")) {
            errors.add("email", new ActionMessage("error.email.invalid"));
        }
        
        if (quantity == null || quantity <= 0) {
            errors.add("quantity", new ActionMessage("error.quantity.invalid"));
        }
        
        return errors;
    }
    
    @Override
    public void reset(ActionMapping mapping, HttpServletRequest request) {
        this.customerName = null;
        this.email = null;
        this.quantity = null;
        this.productId = null;
    }
}
```

**Extraction Notes:**
- Extract all properties (getter/setter pairs) → `FormField` entries
- Extract `validate()` method → `FieldValidation` for each field
- Error keys (e.g., `"error.name.required"`) → i18n message keys
- Extract field types from getter return types
- `reset()` method → form initialization logic

**Migration Path:**
- Convert to POJO / DTO class
- Remove extends ActionForm
- Add Bean Validation annotations:
  - `@NotNull`, `@NotEmpty`, `@NotBlank`
  - `@Email`, `@Pattern`, `@Size`, `@Min`, `@Max`
- Replace `validate()` method with annotations
- Use in Spring controller with `@Valid @ModelAttribute`

---

#### DynaActionForm

|| Pattern | `extends\s+DynaActionForm\b\|type=["']org\.apache\.struts\.action\.DynaActionForm["']` |
||---------|----------------------------------------------------------------------------------------|
|| **File Types** | `*.java`, `struts-config.xml` |
|| **Indicates** | Dynamic form without Java class |
|| **Schema Entity** | `FormDefinition` |

**Example in struts-config.xml:**
```xml
<form-bean name="userForm" type="org.apache.struts.action.DynaActionForm">
    <form-property name="username" type="java.lang.String"/>
    <form-property name="email" type="java.lang.String"/>
    <form-property name="age" type="java.lang.Integer" initial="0"/>
    <form-property name="active" type="java.lang.Boolean" initial="true"/>
</form-bean>
```

**Extraction Notes:**
- Form properties defined in XML instead of Java class
- Each `<form-property>` → `FormField`
- `initial` attribute → default value
- Less type-safe than ActionForm subclasses

**Migration Path:**
- Create explicit POJO class with properties
- Add validation annotations

---

#### ValidatorForm / ValidatorActionForm

|| Pattern | `extends\s+(ValidatorForm\|ValidatorActionForm)\b` |
||---------|-----------------------------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Form with XML-based validation |
|| **Schema Entity** | `FormDefinition` with external validation |

**Example:**
```java
import org.apache.struts.validator.ValidatorForm;

public class UserForm extends ValidatorForm {
    private String username;
    private String email;
    private String password;
    
    // Getters and setters
}
```

**Extraction Notes:**
- Validation rules defined in `validation.xml` instead of `validate()` method
- Form structure extracted same as ActionForm
- Look for corresponding validation XML file

---

### Struts 1.x Validation

#### validation.xml

|| Pattern | `validation\.xml\|validation-rules\.xml` (filename pattern) |
||---------|-------------------------------------------------------------|
|| **File Types** | `validation.xml`, `validation-rules.xml` |
|| **Indicates** | Struts Validator framework configuration |
|| **Schema Entity** | `FieldValidation` |

**Example validation.xml:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE form-validation PUBLIC
    "-//Apache Software Foundation//DTD Commons Validator Rules Configuration 1.3.0//EN"
    "http://jakarta.apache.org/commons/dtds/validator_1_3_0.dtd">
<form-validation>
    <formset>
        <form name="orderForm">
            <field property="customerName" depends="required,maxlength">
                <arg key="label.customer.name" position="0"/>
                <arg name="maxlength" key="${var:maxlength}" position="1"/>
                <var>
                    <var-name>maxlength</var-name>
                    <var-value>100</var-value>
                </var>
            </field>
            
            <field property="email" depends="required,email">
                <arg key="label.email" position="0"/>
            </field>
            
            <field property="quantity" depends="required,integer,minValue">
                <arg key="label.quantity" position="0"/>
                <var>
                    <var-name>minValue</var-name>
                    <var-value>1</var-value>
                </var>
            </field>
        </form>
    </formset>
</form-validation>
```

**Extraction Notes:**
- Each `<form>` → corresponds to form bean by name
- Each `<field>` → `FieldValidation` entry
- `property` → field name
- `depends` → comma-separated list of validators
- `<arg>` → validation error message arguments (i18n keys)
- `<var>` → validator parameters (e.g., max length, min value)

**Common validators:**
- `required` → `@NotNull` / `@NotEmpty`
- `email` → `@Email`
- `maxlength` / `minlength` → `@Size`
- `integer`, `float`, `double` → type validation
- `minValue`, `maxValue` → `@Min`, `@Max`
- `date` → `@Past`, `@Future`, or custom
- `mask` (regex) → `@Pattern`

**Migration Path:**
- Convert to Bean Validation annotations on form/DTO fields
- Custom validators → custom constraint annotations

---

#### validator-rules.xml

|| Pattern | `validator-rules\.xml` (filename pattern) |
||---------|------------------------------------------|
|| **File Types** | `validator-rules.xml` |
|| **Indicates** | Custom validator definitions |
|| **Schema Entity** | Custom validation rules |

**Example:**
```xml
<form-validation>
    <global>
        <validator name="required"
                   classname="org.apache.struts.validator.FieldChecks"
                   method="validateRequired"
                   methodParams="java.lang.Object,
                                 org.apache.commons.validator.ValidatorAction,
                                 org.apache.commons.validator.Field,
                                 org.apache.struts.action.ActionMessages,
                                 org.apache.commons.validator.Validator,
                                 javax.servlet.http.HttpServletRequest"
                   msg="errors.required"/>
    </global>
</form-validation>
```

**Extraction Notes:**
- Defines built-in and custom validator implementations
- Reference for understanding validation.xml rules
- Custom validators need manual migration to Bean Validation

---

### Struts 1.x Tag Libraries

#### Struts HTML Tag Library

|| Pattern | `<%@\s*taglib.*uri=["']/tags/struts-html["']` |
||---------|-----------------------------------------------|
|| **File Types** | `*.jsp` |
|| **Indicates** | Struts HTML form tags |
|| **Schema Entity** | `ViewRequirement.tag_libraries` with `type = STRUTS_HTML` |

**Example:**
```jsp
<%@ taglib prefix="html" uri="/tags/struts-html" %>

<html:form action="/orders/create" method="POST">
    <html:text property="customerName" size="50" maxlength="100"/>
    <html:errors property="customerName"/>
    
    <html:text property="email" size="50"/>
    <html:errors property="email"/>
    
    <html:select property="productId">
        <html:option value="">-- Select Product --</html:option>
        <html:options collection="products" property="id" labelProperty="name"/>
    </html:select>
    
    <html:text property="quantity" size="5"/>
    
    <html:checkbox property="expedited"/>
    
    <html:submit value="Create Order"/>
    <html:cancel value="Cancel"/>
</html:form>

<html:link action="/orders/list">Back to List</html:link>
<html:img src="/images/logo.png" alt="Logo"/>
```

**Common Struts HTML Tags:**
- `<html:form>` → form with action mapping
- `<html:text>`, `<html:password>` → text inputs
- `<html:textarea>` → multi-line text
- `<html:checkbox>`, `<html:radio>` → boolean/selection inputs
- `<html:select>`, `<html:option>`, `<html:options>` → dropdowns
- `<html:submit>`, `<html:cancel>`, `<html:reset>` → buttons
- `<html:hidden>` → hidden fields
- `<html:errors>` → validation error display
- `<html:link>` → action links
- `<html:img>`, `<html:base>` → HTML elements

**Extraction Notes:**
- Extract form structure → `FormDefinition`
- `property` attributes → `FormField.name`
- `<html:errors>` placement → validation error display requirements

**Migration Path:**
- `<html:form>` → Thymeleaf `<form th:action="@{/path}" th:object="${formObject}">`
- `<html:text property="x">` → `<input type="text" th:field="*{x}" />`
- `<html:select>` → `<select th:field="*{x}"><option th:each="..."></option></select>`
- `<html:errors>` → `<span th:if="${#fields.hasErrors('x')}" th:errors="*{x}"></span>`

---

#### Struts Bean Tag Library

|| Pattern | `<%@\s*taglib.*uri=["']/tags/struts-bean["']` |
||---------|-----------------------------------------------|
|| **File Types** | `*.jsp` |
|| **Indicates** | Struts bean manipulation tags |
|| **Schema Entity** | `ViewRequirement.tag_libraries` with `type = STRUTS_BEAN` |

**Example:**
```jsp
<%@ taglib prefix="bean" uri="/tags/struts-bean" %>

<!-- Display bean property -->
<bean:write name="order" property="orderNumber"/>

<!-- Display with formatting -->
<bean:write name="order" property="total" format="$#,##0.00"/>

<!-- Display i18n message -->
<bean:message key="label.order.number"/>
<bean:message key="label.welcome" arg0="${user.name}"/>

<!-- Bean operations -->
<bean:define id="status" name="order" property="status"/>
<bean:size id="itemCount" name="order" property="items"/>

<!-- Include another page -->
<bean:include id="header" page="/includes/header.jsp"/>
```

**Common Struts Bean Tags:**
- `<bean:write>` → output bean property
- `<bean:message>` → i18n message lookup
- `<bean:define>` → create page-scoped variable
- `<bean:size>` → get collection size
- `<bean:include>` → include content
- `<bean:cookie>`, `<bean:header>`, `<bean:parameter>` → request data access

**Migration Path:**
- `<bean:write>` → Thymeleaf `<span th:text="${bean.property}"></span>`
- `<bean:message>` → Thymeleaf `#{message.key}` or `<span th:text="#{key}"></span>`
- `<bean:define>` → Thymeleaf `th:with="var=${value}"`

---

#### Struts Logic Tag Library

|| Pattern | `<%@\s*taglib.*uri=["']/tags/struts-logic["']` |
||---------|------------------------------------------------|
|| **File Types** | `*.jsp` |
|| **Indicates** | Struts conditional/iteration tags |
|| **Schema Entity** | `ViewRequirement.tag_libraries` with `type = STRUTS_LOGIC` |

**Example:**
```jsp
<%@ taglib prefix="logic" uri="/tags/struts-logic" %>

<!-- Iteration -->
<logic:iterate id="order" name="orders">
    <tr>
        <td><bean:write name="order" property="orderNumber"/></td>
        <td><bean:write name="order" property="status"/></td>
    </tr>
</logic:iterate>

<!-- Conditionals -->
<logic:present name="user">
    Welcome, <bean:write name="user" property="username"/>
</logic:present>

<logic:notPresent name="user">
    <a href="/login.do">Please log in</a>
</logic:notPresent>

<logic:equal name="order" property="status" value="PENDING">
    <button>Cancel Order</button>
</logic:equal>

<logic:notEqual name="order" property="status" value="CANCELLED">
    <button>View Details</button>
</logic:notEqual>

<logic:greaterThan name="order" property="total" value="100">
    <span class="high-value">High Value Order</span>
</logic:greaterThan>

<!-- Redirects -->
<logic:redirect href="/error.jsp"/>
<logic:forward name="success"/>
```

**Common Struts Logic Tags:**
- `<logic:iterate>` → loop over collection
- `<logic:present>`, `<logic:notPresent>` → check object existence
- `<logic:equal>`, `<logic:notEqual>` → equality checks
- `<logic:greaterThan>`, `<logic:lessThan>` → numeric comparisons
- `<logic:match>`, `<logic:notMatch>` → string matching
- `<logic:empty>`, `<logic:notEmpty>` → empty checks
- `<logic:redirect>`, `<logic:forward>` → navigation

**Migration Path:**
- `<logic:iterate>` → Thymeleaf `<tr th:each="order : ${orders}">`
- `<logic:present>` → Thymeleaf `<div th:if="${user != null}">`
- `<logic:equal>` → Thymeleaf `<div th:if="${order.status == 'PENDING'}">`
- Logic tags generally → Thymeleaf `th:if`, `th:unless`, `th:each`, `th:switch`

---

### Struts 1.x Tiles Integration

#### tiles-defs.xml (Struts 1.x)

|| Pattern | `tiles-defs\.xml` (filename pattern with Struts 1.x DTD) |
||---------|----------------------------------------------------------|
|| **File Types** | `tiles-defs.xml` |
|| **Indicates** | Tiles 1.x layout framework for Struts 1.x |
|| **Schema Entity** | `ViewRequirement.layout_framework = TILES`, `ViewComponent` |

**Example:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE tiles-definitions PUBLIC
    "-//Apache Software Foundation//DTD Tiles Configuration 1.1//EN"
    "http://jakarta.apache.org/struts/dtds/tiles-config_1_1.dtd">
<tiles-definitions>
    <!-- Base layout -->
    <definition name="baseLayout" path="/WEB-INF/layouts/base.jsp">
        <put name="title" value="My Application"/>
        <put name="header" value="/WEB-INF/includes/header.jsp"/>
        <put name="menu" value="/WEB-INF/includes/menu.jsp"/>
        <put name="footer" value="/WEB-INF/includes/footer.jsp"/>
    </definition>
    
    <!-- Page extending base layout -->
    <definition name="orderList" extends="baseLayout">
        <put name="title" value="Order List"/>
        <put name="body" value="/WEB-INF/pages/orders/list.jsp"/>
    </definition>
    
    <definition name="orderDetail" extends="baseLayout">
        <put name="title" value="Order Detail"/>
        <put name="body" value="/WEB-INF/pages/orders/detail.jsp"/>
    </definition>
</tiles-definitions>
```

**Extraction Notes:**
- Each `<definition>` → `ViewComponent` with `type = LAYOUT`
- `extends` → layout inheritance hierarchy
- `<put>` → tile attribute (header, footer, body, etc.)
- Actions forward to definition names instead of JSP paths

**Migration Path:**
- Convert to Thymeleaf Layout Dialect
- Base layout → layout template with `layout:fragment` sections
- Page definitions → content pages with `layout:decorate`

---

#### Tiles Plugin Configuration

|| Pattern | `<plug-in\s+className=["']org\.apache\.struts\.tiles\.TilesPlugin["']` |
||---------|------------------------------------------------------------------------|
|| **File Types** | `struts-config.xml` |
|| **Indicates** | Tiles plugin enabled |
|| **Schema Entity** | `ViewRequirement.layout_framework = TILES` |

**Example:**
```xml
<plug-in className="org.apache.struts.tiles.TilesPlugin">
    <set-property property="definitions-config" 
                  value="/WEB-INF/tiles-defs.xml,/WEB-INF/tiles-orders.xml"/>
    <set-property property="moduleAware" value="true"/>
    <set-property property="definitions-parser-validate" value="true"/>
</plug-in>
```

---

#### Tiles JSP Tags (Struts 1.x)

|| Pattern | `<%@\s*taglib.*uri=["']/tags/struts-tiles["']` |
||---------|------------------------------------------------|
|| **File Types** | `*.jsp` |
|| **Indicates** | Tiles tag usage in JSP |
|| **Schema Entity** | `ViewRequirement.tag_libraries` with `type = TILES` |

**Example:**
```jsp
<%@ taglib prefix="tiles" uri="/tags/struts-tiles" %>

<!-- Insert tile definition -->
<tiles:insert definition="orderList"/>

<!-- Insert with attributes -->
<tiles:insert definition="baseLayout">
    <tiles:put name="title" value="Custom Page"/>
    <tiles:put name="body" value="/WEB-INF/pages/custom.jsp"/>
</tiles:insert>

<!-- Get tile attribute -->
<tiles:getAsString name="title"/>

<!-- Insert specific tile component -->
<tiles:insert attribute="header"/>
<tiles:insert attribute="body"/>
<tiles:insert attribute="footer"/>
```

---

## Struts 2.x Patterns

### Struts 2.x Configuration Files

#### struts.xml

|| Pattern | `struts\.xml` (filename pattern) |
||---------|----------------------------------|
|| **File Types** | `struts.xml` |
|| **Indicates** | Struts 2.x application configuration |
|| **Schema Entity** | `ModernizationRequirements.source_technology = STRUTS_2` |

**Example:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE struts PUBLIC
    "-//Apache Software Foundation//DTD Struts Configuration 2.5//EN"
    "http://struts.apache.org/dtds/struts-2.5.dtd">
<struts>
    <constant name="struts.devMode" value="false"/>
    <constant name="struts.ui.theme" value="simple"/>
    <constant name="struts.action.extension" value="action"/>
    
    <package name="default" namespace="/" extends="struts-default">
        <interceptors>
            <interceptor name="authInterceptor" class="com.example.AuthInterceptor"/>
            <interceptor-stack name="secureStack">
                <interceptor-ref name="authInterceptor"/>
                <interceptor-ref name="defaultStack"/>
            </interceptor-stack>
        </interceptors>
        
        <default-interceptor-ref name="secureStack"/>
        
        <global-results>
            <result name="error">/WEB-INF/pages/error.jsp</result>
            <result name="login" type="redirectAction">login</result>
        </global-results>
        
        <global-exception-mappings>
            <exception-mapping exception="java.lang.Exception" result="error"/>
        </global-exception-mappings>
        
        <action name="login" class="com.example.actions.LoginAction">
            <result name="success">/WEB-INF/pages/home.jsp</result>
            <result name="input">/WEB-INF/pages/login.jsp</result>
        </action>
    </package>
    
    <package name="orders" namespace="/orders" extends="default">
        <action name="list" class="com.example.actions.OrderAction" method="list">
            <result>/WEB-INF/pages/orders/list.jsp</result>
        </action>
        
        <action name="view" class="com.example.actions.OrderAction" method="view">
            <result>/WEB-INF/pages/orders/detail.jsp</result>
        </action>
        
        <action name="create" class="com.example.actions.OrderAction" method="create">
            <result name="success" type="redirectAction">
                <param name="actionName">list</param>
            </result>
            <result name="input">/WEB-INF/pages/orders/form.jsp</result>
        </action>
        
        <action name="delete" class="com.example.actions.OrderAction" method="delete">
            <result type="redirectAction">list</result>
        </action>
    </package>
</struts>
```

**Extraction Notes:**
- Parse `<constant>` → framework configuration
- Parse `<package>` → namespace grouping
  - `namespace` → URL prefix for all actions in package
  - `extends` → parent package (typically `"struts-default"`)
- Parse `<action>` → `ServiceEndpoint`
  - `name` → action name (part of URL)
  - `class` → action class
  - `method` → specific method in action class
  - URL pattern: `/{namespace}/{action-name}.{extension}`
- Parse `<result>` → navigation outcome
  - `name` → result code (default: "success")
  - `type` → result type (dispatcher, redirect, redirectAction, stream, etc.)
  - content or `<param>` → target view/action
- Parse `<interceptor>` and `<interceptor-stack>` → cross-cutting concerns
- Parse `<global-results>` → shared navigation
- Parse `<global-exception-mappings>` → error handling

**Migration Path:**
- Each action → Spring `@Controller` method
- Namespaces → Spring `@RequestMapping` at class level
- Results → Spring view names or `RedirectView`
- Interceptors → Spring AOP `@Aspect` or servlet `Filter`

---

#### Package Configuration

|| Pattern | `<package.*extends=["']struts-default["']` |
||---------|---------------------------------------------|
|| **File Types** | `struts.xml` |
|| **Indicates** | Struts 2.x action package |
|| **Schema Entity** | Grouping for `ServiceEndpoint` |

**Extraction Notes:**
- Packages group related actions
- `namespace` attribute → URL path prefix
- Actions inherit interceptors and settings from parent package

---

#### Action Configuration

|| Pattern | `<action\s+name=["'][^"']+["']\s+class=["'][^"']+["']` |
||---------|--------------------------------------------------------|
|| **File Types** | `struts.xml` |
|| **Indicates** | Struts 2.x action mapping |
|| **Schema Entity** | `ServiceEndpoint` |

**Extraction Notes:**
- `name` + package `namespace` → full URL path
- `class` → action implementation
- `method` → specific method (default: `execute`)
- Multiple `<result>` elements → possible outcomes

---

#### Result Types

Common result types in Struts 2.x:

- **dispatcher** (default) → forward to JSP
- **redirect** → HTTP redirect to URL
- **redirectAction** → redirect to another action
- **stream** → binary content (file download)
- **json** → JSON response (with JSON plugin)
- **freemarker**, **velocity** → template engines
- **tiles** → Tiles framework (with Tiles plugin)
- **chain** → chain to another action

---

### Struts 2.x Annotation-Based Configuration

#### @Action Annotation

|| Pattern | `@Action\s*\(` |
||---------|----------------|
|| **File Types** | `*.java` |
|| **Indicates** | Annotation-based action configuration |
|| **Schema Entity** | `ServiceEndpoint` |

**Example:**
```java
import org.apache.struts2.convention.annotation.Action;
import org.apache.struts2.convention.annotation.Result;
import org.apache.struts2.convention.annotation.Namespace;
import org.apache.struts2.convention.annotation.ParentPackage;

@Namespace("/orders")
@ParentPackage("default")
public class OrderAction extends ActionSupport {
    
    private List<Order> orders;
    private Order order;
    private Long orderId;
    
    @Action(value = "list", results = {
        @Result(name = "success", location = "/WEB-INF/pages/orders/list.jsp")
    })
    public String list() {
        orders = orderService.findAll();
        return SUCCESS;
    }
    
    @Action(value = "view", results = {
        @Result(name = "success", location = "/WEB-INF/pages/orders/detail.jsp"),
        @Result(name = "error", location = "/WEB-INF/pages/error.jsp")
    })
    public String view() {
        if (orderId == null) {
            return ERROR;
        }
        order = orderService.findById(orderId);
        return SUCCESS;
    }
    
    @Action(value = "create", results = {
        @Result(name = "success", type = "redirectAction", location = "list"),
        @Result(name = "input", location = "/WEB-INF/pages/orders/form.jsp")
    })
    public String create() {
        orderService.save(order);
        return SUCCESS;
    }
    
    // Getters and setters for properties
}
```

**Extraction Notes:**
- `@Action(value="name")` → action name (URL component)
- `results` → array of `@Result` annotations
- Method return value → result code ("success", "error", "input", etc.)
- Class-level `@Namespace` → URL prefix
- Properties with getters/setters → request parameters / form fields

**Migration Path:**
- `@Action` → Spring `@GetMapping` or `@PostMapping`
- Action properties → `@RequestParam` or `@ModelAttribute` in Spring
- Result codes → return view name or `RedirectView`

---

#### @Result Annotation

|| Pattern | `@Result\s*\(` |
||---------|----------------|
|| **File Types** | `*.java` |
|| **Indicates** | Action result configuration |
|| **Schema Entity** | `NavigationFlow` |

**Extraction Notes:**
- Can be method-level (inside `@Action`) or class-level (global to action)
- `name` → result code
- `location` → target view/action
- `type` → result type (default: dispatcher)
- `params` → additional parameters for result

---

#### @Namespace Annotation

|| Pattern | `@Namespace\s*\(` |
||---------|-------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Action namespace (URL prefix) |
|| **Schema Entity** | URL path prefix for endpoints |

---

#### @ParentPackage Annotation

|| Pattern | `@ParentPackage\s*\(` |
||---------|----------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Extends Struts package configuration |
|| **Schema Entity** | Metadata (inherits interceptors/settings) |

---

#### @InterceptorRef Annotation

|| Pattern | `@InterceptorRef\s*\(` |
||---------|------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Apply interceptor to action |
|| **Schema Entity** | Cross-cutting concern |

**Example:**
```java
@Action(value = "secureAction", 
        interceptorRefs = {
            @InterceptorRef("authInterceptor"),
            @InterceptorRef("defaultStack")
        })
public String secureAction() {
    // Protected action logic
    return SUCCESS;
}
```

---

### Struts 2.x Action Classes

#### ActionSupport Class

|| Pattern | `extends\s+ActionSupport\b` |
||---------|------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Struts 2.x action with validation/i18n support |
|| **Schema Entity** | `ServiceEndpoint`, `ServiceOperation` |

**Example:**
```java
import com.opensymphony.xwork2.ActionSupport;

public class OrderAction extends ActionSupport {
    
    private OrderService orderService;
    private Order order;
    private Long orderId;
    
    // Action is instantiated per request (not singleton like Struts 1.x)
    
    public String execute() {
        // Default method
        return SUCCESS;
    }
    
    public String list() {
        List<Order> orders = orderService.findAll();
        // Orders available to view via getValue Stack
        return SUCCESS;
    }
    
    public String view() {
        order = orderService.findById(orderId);
        return SUCCESS;
    }
    
    public String create() {
        if (hasErrors()) {
            return INPUT;
        }
        orderService.save(order);
        addActionMessage("Order created successfully");
        return SUCCESS;
    }
    
    @Override
    public void validate() {
        if (order != null) {
            if (order.getCustomerName() == null || order.getCustomerName().trim().isEmpty()) {
                addFieldError("order.customerName", "Customer name is required");
            }
            if (order.getQuantity() == null || order.getQuantity() <= 0) {
                addFieldError("order.quantity", "Quantity must be greater than zero");
            }
        }
    }
    
    // Getters and setters
    public Order getOrder() { return order; }
    public void setOrder(Order order) { this.order = order; }
    
    public Long getOrderId() { return orderId; }
    public void setOrderId(Long orderId) { this.orderId = orderId; }
}
```

**Extraction Notes:**
- Each public method returning `String` → potential action method (`ServiceEndpoint`)
- Properties (getters/setters) → request parameters, form fields, or view model
- `validate()` method → field validation
- `addFieldError()` → validation error messages
- `addActionMessage()` / `addActionError()` → user feedback messages
- Per-request instantiation → thread-safe by design

**ActionSupport convenience methods:**
- `getText(String key)` → i18n message lookup
- `hasErrors()` → check for validation errors
- `addFieldError()`, `addActionError()`, `addActionMessage()`

**Result codes (constants in ActionSupport):**
- `SUCCESS` → "success"
- `ERROR` → "error"
- `INPUT` → "input" (validation failure)
- `LOGIN` → "login"
- `NONE` → null (no result)

**Migration Path:**
- Convert to Spring `@Controller`
- Methods → `@GetMapping` / `@PostMapping` methods
- `validate()` → Bean Validation annotations
- Properties → method parameters with `@RequestParam` / `@ModelAttribute`
- Result codes → view names

---

#### Action Interface

|| Pattern | `implements\s+Action\b` |
||---------|-------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Minimal Struts 2.x action |
|| **Schema Entity** | `ServiceEndpoint` |

**Example:**
```java
import com.opensymphony.xwork2.Action;

public class SimpleAction implements Action {
    
    @Override
    public String execute() throws Exception {
        // Action logic
        return SUCCESS;
    }
}
```

**Extraction Notes:**
- Minimal interface, only defines constants
- Less common than extending ActionSupport

---

#### POJO Action (Convention-Based)

|| Pattern | `execute\s*\(\s*\)\s*throws\s+Exception` (method signature) |
||---------|-------------------------------------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | POJO action (no extends/implements) |
|| **Schema Entity** | `ServiceEndpoint` |

**Example:**
```java
package com.example.action;

public class HelloAction {
    private String message;
    
    public String execute() {
        message = "Hello, World!";
        return "success";
    }
    
    public String getMessage() { return message; }
    public void setMessage(String message) { this.message = message; }
}
```

**Extraction Notes:**
- Any class with `execute()` method can be an action
- Requires Convention Plugin or explicit configuration
- Package must follow naming convention (e.g., `*.action` package)

---

#### ActionContext Usage

|| Pattern | `ActionContext\.getContext\(\)` |
||---------|----------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Accessing servlet context in Struts 2.x |
|| **Schema Entity** | Metadata |

**Example:**
```java
import com.opensymphony.xwork2.ActionContext;

ActionContext context = ActionContext.getContext();
Map<String, Object> session = context.getSession();
HttpServletRequest request = ServletActionContext.getRequest();
```

**Extraction Notes:**
- Struts 2.x abstracts servlet API via ActionContext
- ValueStack access for request/session/application scopes

---

### Struts 2.x Convention Plugin

#### Convention Plugin Package Detection

|| Pattern | Package name ending with `.action` or `.actions` |
||---------|--------------------------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Convention Plugin action discovery |
|| **Schema Entity** | `ServiceEndpoint` |

**Example:**
```java
package com.example.struts.action;

public class OrderAction {
    public String execute() {
        return "success";
    }
}
```

**Extraction Notes:**
- Convention Plugin auto-discovers actions by package name
- Default result location: `/WEB-INF/content/{action-name}-{result}.jsp`
- Action name derived from class name (e.g., `OrderAction` → `order`)

---

#### Convention Plugin Result Location

|| Pattern | `/WEB-INF/content/{action}-{result}.jsp` (file path pattern) |
||---------|--------------------------------------------------------------|
|| **File Types** | `*.jsp` |
|| **Indicates** | Convention-based view resolution |
|| **Schema Entity** | `ViewComponent` |

**Example file structure:**
```
/WEB-INF/content/
    order-success.jsp
    order-input.jsp
    order-error.jsp
```

---

### Struts 2.x Validation Framework

#### XML Validation File (Struts 2.x)

|| Pattern | `{ActionClassName}-validation\.xml` (filename pattern) |
||---------|--------------------------------------------------------|
|| **File Types** | `*-validation.xml` |
|| **Indicates** | Struts 2.x XML-based validation |
|| **Schema Entity** | `FieldValidation` |

**Example OrderAction-validation.xml:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE validators PUBLIC
    "-//Apache Struts//XWork Validator 1.0.3//EN"
    "http://struts.apache.org/dtds/xwork-validator-1.0.3.dtd">
<validators>
    <field name="order.customerName">
        <field-validator type="requiredstring">
            <message>Customer name is required</message>
        </field-validator>
        <field-validator type="stringlength">
            <param name="minLength">2</param>
            <param name="maxLength">100</param>
            <message>Customer name must be between 2 and 100 characters</message>
        </field-validator>
    </field>
    
    <field name="order.email">
        <field-validator type="requiredstring">
            <message>Email is required</message>
        </field-validator>
        <field-validator type="email">
            <message>Invalid email format</message>
        </field-validator>
    </field>
    
    <field name="order.quantity">
        <field-validator type="required">
            <message>Quantity is required</message>
        </field-validator>
        <field-validator type="int">
            <param name="min">1</param>
            <param name="max">1000</param>
            <message>Quantity must be between 1 and 1000</message>
        </field-validator>
    </field>
</validators>
```

**Extraction Notes:**
- File name pattern: `{ActionClassName}-validation.xml`
- For action method validation: `{ActionClassName}-{actionName}-validation.xml`
- Each `<field>` → field name (can use OGNL expression like `order.customerName`)
- `<field-validator type="">` → validator type
- `<param>` → validator parameters
- `<message>` → error message

**Common validator types:**
- `required` → not null
- `requiredstring` → not empty string
- `stringlength` → min/max length
- `email` → email format
- `url` → URL format
- `int`, `double`, `date` → type validation
- `regex` → regular expression
- `fieldexpression` → OGNL expression validation
- `visitor` → nested object validation

**Migration Path:**
- Convert to Bean Validation annotations
- Custom validators → custom constraint annotations

---

#### Validation Annotations (Struts 2.x)

|| Pattern | `@Validations\s*\(` |
||---------|---------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Annotation-based validation |
|| **Schema Entity** | `FieldValidation` |

**Example:**
```java
import org.apache.struts2.convention.annotation.Action;
import com.opensymphony.xwork2.validator.annotations.*;

public class OrderAction extends ActionSupport {
    
    private Order order;
    
    @Validations(
        requiredStrings = {
            @RequiredStringValidator(fieldName = "order.customerName", 
                                     message = "Customer name is required")
        },
        emails = {
            @EmailValidator(fieldName = "order.email", 
                           message = "Invalid email format")
        },
        intRangeFields = {
            @IntRangeFieldValidator(fieldName = "order.quantity", 
                                   min = "1", max = "1000", 
                                   message = "Quantity must be between 1 and 1000")
        }
    )
    @Action(value = "create")
    public String create() {
        orderService.save(order);
        return SUCCESS;
    }
}
```

**Common validation annotations:**
- `@RequiredFieldValidator`
- `@RequiredStringValidator`
- `@StringLengthFieldValidator`
- `@EmailValidator`
- `@UrlValidator`
- `@IntRangeFieldValidator`, `@DoubleRangeFieldValidator`
- `@DateRangeFieldValidator`
- `@RegexFieldValidator`
- `@FieldExpressionValidator`

---

#### validate() Method (Struts 2.x)

|| Pattern | `public\s+void\s+validate\s*\(\s*\)` |
||---------|--------------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Programmatic validation in action |
|| **Schema Entity** | `FieldValidation` |

**Example:**
```java
@Override
public void validate() {
    if (order == null) {
        addActionError("Order object is required");
        return;
    }
    
    if (order.getCustomerName() == null || order.getCustomerName().trim().isEmpty()) {
        addFieldError("order.customerName", "Customer name is required");
    }
    
    if (order.getEmail() != null && !order.getEmail().matches("^[A-Za-z0-9+_.-]+@(.+)$")) {
        addFieldError("order.email", "Invalid email format");
    }
}
```

**Extraction Notes:**
- `addFieldError(fieldName, message)` → field-specific error
- `addActionError(message)` → general error
- Called before action method if validation succeeds
- If errors exist, action method not called, returns INPUT result

---

### Struts 2.x Interceptors

#### Interceptor Interface

|| Pattern | `implements\s+Interceptor\b` |
||---------|------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Custom interceptor implementation |
|| **Schema Entity** | Cross-cutting concern |

**Example:**
```java
import com.opensymphony.xwork2.ActionInvocation;
import com.opensymphony.xwork2.interceptor.Interceptor;

public class AuthInterceptor implements Interceptor {
    
    @Override
    public void init() {
        // Initialization
    }
    
    @Override
    public void destroy() {
        // Cleanup
    }
    
    @Override
    public String intercept(ActionInvocation invocation) throws Exception {
        // Pre-processing
        Map<String, Object> session = invocation.getInvocationContext().getSession();
        User user = (User) session.get("user");
        
        if (user == null) {
            return "login";
        }
        
        // Continue to action
        String result = invocation.invoke();
        
        // Post-processing
        return result;
    }
}
```

**Extraction Notes:**
- `intercept()` method contains logic
- `invocation.invoke()` → proceed to next interceptor or action
- Can modify request/response, short-circuit execution, or add behavior
- **Thread-safety:** Interceptors are shared between requests (like Struts 1.x Actions)

**Migration Path:**
- Convert to Spring AOP `@Aspect` with `@Around` advice
- Or implement as servlet `Filter`
- Or use Spring Security for authentication/authorization

---

#### AbstractInterceptor Class

|| Pattern | `extends\s+AbstractInterceptor\b` |
||---------|-----------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Simplified interceptor (no init/destroy) |
|| **Schema Entity** | Cross-cutting concern |

**Example:**
```java
import com.opensymphony.xwork2.ActionInvocation;
import com.opensymphony.xwork2.interceptor.AbstractInterceptor;

public class LoggingInterceptor extends AbstractInterceptor {
    
    @Override
    public String intercept(ActionInvocation invocation) throws Exception {
        long startTime = System.currentTimeMillis();
        
        String result = invocation.invoke();
        
        long endTime = System.currentTimeMillis();
        System.out.println("Action took " + (endTime - startTime) + "ms");
        
        return result;
    }
}
```

---

#### Interceptor Configuration in struts.xml

|| Pattern | `<interceptor\s+name=["'][^"']+["']\s+class=["'][^"']+["']` |
||---------|-------------------------------------------------------------|
|| **File Types** | `struts.xml` |
|| **Indicates** | Interceptor registration |
|| **Schema Entity** | Cross-cutting concern |

**Example:**
```xml
<interceptors>
    <interceptor name="auth" class="com.example.AuthInterceptor"/>
    <interceptor name="logging" class="com.example.LoggingInterceptor"/>
    
    <interceptor-stack name="secureStack">
        <interceptor-ref name="logging"/>
        <interceptor-ref name="auth"/>
        <interceptor-ref name="defaultStack"/>
    </interceptor-stack>
</interceptors>

<default-interceptor-ref name="secureStack"/>
```

**Extraction Notes:**
- `<interceptor>` → register custom interceptor
- `<interceptor-stack>` → chain multiple interceptors
- `<interceptor-ref>` → apply interceptor to action
- `<default-interceptor-ref>` → default stack for all actions in package
- Order matters: interceptors execute in defined order

**Built-in interceptors (from defaultStack):**
- `params` → parameter binding
- `validation` → validation execution
- `workflow` → handles validation failure
- `servlet-config` → inject servlet objects
- `prepare` → prepare() method invocation
- `model-driven` → model-driven action support
- `fileUpload` → multipart file upload
- `exception` → exception handling

---

### Struts 2.x Tag Libraries

#### Struts 2 Tag Library

|| Pattern | `<%@\s*taglib.*uri=["']/struts-tags["']` |
||---------|------------------------------------------|
|| **File Types** | `*.jsp` |
|| **Indicates** | Struts 2.x tag library |
|| **Schema Entity** | `ViewRequirement.tag_libraries` with `type = STRUTS_2` |

**Example:**
```jsp
<%@ taglib prefix="s" uri="/struts-tags" %>

<!-- Form -->
<s:form action="createOrder" method="POST">
    <s:textfield name="order.customerName" label="Customer Name" required="true"/>
    <s:textfield name="order.email" label="Email"/>
    <s:select name="order.productId" label="Product" 
              list="products" listKey="id" listValue="name"/>
    <s:textfield name="order.quantity" label="Quantity"/>
    <s:checkbox name="order.expedited" label="Expedited Shipping"/>
    <s:submit value="Create Order"/>
</s:form>

<!-- Data Display -->
<s:property value="order.orderNumber"/>
<s:property value="order.total" />

<!-- Iteration -->
<table>
    <s:iterator value="orders" var="order">
        <tr>
            <td><s:property value="#order.orderNumber"/></td>
            <td><s:property value="#order.status"/></td>
            <td><s:property value="#order.total"/></td>
            <td>
                <s:url action="view" var="viewUrl">
                    <s:param name="orderId" value="#order.id"/>
                </s:url>
                <s:a href="%{viewUrl}">View</s:a>
            </td>
        </tr>
    </s:iterator>
</table>

<!-- Conditionals -->
<s:if test="order.status == 'PENDING'">
    <button>Cancel Order</button>
</s:if>
<s:elseif test="order.status == 'SHIPPED'">
    <span>In Transit</span>
</s:elseif>
<s:else>
    <span>Completed</span>
</s:else>

<!-- i18n -->
<s:text name="label.welcome"/>
<s:text name="label.greeting">
    <s:param value="user.name"/>
</s:text>

<!-- URL Generation -->
<s:url action="orderList" var="listUrl"/>
<s:a href="%{listUrl}">View All Orders</s:a>
```

**Common Struts 2 Tags:**

**UI Tags:**
- `<s:form>`, `<s:textfield>`, `<s:password>`, `<s:textarea>`
- `<s:select>`, `<s:radio>`, `<s:checkbox>`, `<s:checkboxlist>`
- `<s:submit>`, `<s:reset>`
- `<s:hidden>`, `<s:file>`, `<s:label>`
- `<s:fielderror>` → display validation errors

**Data Tags:**
- `<s:property>` → display value
- `<s:set>` → create variable
- `<s:push>` → push value onto ValueStack
- `<s:bean>` → create bean
- `<s:param>` → parameter for parent tag

**Control Tags:**
- `<s:if>`, `<s:elseif>`, `<s:else>` → conditionals
- `<s:iterator>` → loop over collection
- `<s:merge>`, `<s:append>`, `<s:subset>` → collection manipulation

**URL Tags:**
- `<s:url>` → build URL
- `<s:a>` → anchor link
- `<s:form>` → form with action

**i18n Tags:**
- `<s:text>` → message lookup
- `<s:i18n>` → set resource bundle

**OGNL Expressions:**
- `%{expression}` → forced evaluation
- `${expression}` → deferred evaluation
- `#variable` → ValueStack variable reference

**Extraction Notes:**
- Extract form structure → `FormDefinition`
- Extract field names → `FormField`
- Extract URLs and actions → `ServiceEndpoint`
- Extract iteration patterns → list data requirements
- Extract i18n keys → internationalization requirements

**Migration Path:**
- `<s:form>` → Thymeleaf `<form th:action="@{/path}" th:object="${model}">`
- `<s:textfield name="x">` → `<input type="text" th:field="*{x}" />`
- `<s:select>` → `<select th:field="*{x}"><option th:each="..."></option></select>`
- `<s:property value="x"/>` → `<span th:text="${x}"></span>`
- `<s:iterator>` → `<tr th:each="item : ${items}">`
- `<s:if>` → `<div th:if="${condition}">`
- `<s:text>` → `<span th:text="#{key}"></span>`
- OGNL expressions → Thymeleaf variable expressions `${...}`

---

### Struts 2.x Tiles Integration

#### tiles.xml (Tiles 2.x/3.x for Struts 2)

|| Pattern | `tiles\.xml` (filename with Tiles 2/3 namespace) |
||---------|--------------------------------------------------|
|| **File Types** | `tiles.xml` |
|| **Indicates** | Tiles 2.x or 3.x layout framework |
|| **Schema Entity** | `ViewRequirement.layout_framework = TILES`, `ViewComponent` |

**Example:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE tiles-definitions PUBLIC
    "-//Apache Software Foundation//DTD Tiles Configuration 3.0//EN"
    "http://tiles.apache.org/dtds/tiles-config_3_0.dtd">
<tiles-definitions>
    <definition name="baseLayout" template="/WEB-INF/layouts/base.jsp">
        <put-attribute name="title" value="My Application"/>
        <put-attribute name="header" value="/WEB-INF/includes/header.jsp"/>
        <put-attribute name="menu" value="/WEB-INF/includes/menu.jsp"/>
        <put-attribute name="body" value=""/>
        <put-attribute name="footer" value="/WEB-INF/includes/footer.jsp"/>
    </definition>
    
    <definition name="orders.list" extends="baseLayout">
        <put-attribute name="title" value="Order List"/>
        <put-attribute name="body" value="/WEB-INF/pages/orders/list.jsp"/>
    </definition>
    
    <definition name="orders.detail" extends="baseLayout">
        <put-attribute name="title" value="Order Detail"/>
        <put-attribute name="body" value="/WEB-INF/pages/orders/detail.jsp"/>
    </definition>
</tiles-definitions>
```

**Extraction Notes:**
- Similar to Struts 1.x Tiles but uses Tiles 2/3 DTD
- Each `<definition>` → `ViewComponent` with layout
- Actions return definition names as results

---

#### Tiles Annotations (Struts 2.x)

|| Pattern | `@TilesDefinition\s*\(` |
||---------|-------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Annotation-based Tiles configuration |
|| **Schema Entity** | `ViewComponent` |

**Example:**
```java
import org.apache.tiles.annotation.TilesDefinition;
import org.apache.tiles.annotation.TilesPutAttribute;
import org.apache.struts2.convention.annotation.Result;

@TilesDefinition(
    name = "orderDetail",
    extend = "baseLayout",
    putAttributes = {
        @TilesPutAttribute(name = "title", value = "Order Detail"),
        @TilesPutAttribute(name = "body", value = "/WEB-INF/pages/orders/detail.jsp")
    }
)
@Result(name = "success", type = "tiles", location = "orderDetail")
public String view() {
    return SUCCESS;
}
```

**Extraction Notes:**
- Defines Tiles definitions inline with actions
- Reduces XML configuration

---

#### Tiles Result Type

|| Pattern | `type=["']tiles["']` |
||---------|----------------------|
|| **File Types** | `struts.xml`, `*.java` |
|| **Indicates** | Action returns Tiles definition |
|| **Schema Entity** | `ViewComponent` |

**Example struts.xml:**
```xml
<action name="orderList" class="com.example.OrderAction" method="list">
    <result name="success" type="tiles">orders.list</result>
</action>
```

---

#### StrutsTilesListener

|| Pattern | `org\.apache\.struts2\.tiles\.StrutsTilesListener` |
||---------|-----------------------------------------------------|
|| **File Types** | `web.xml` |
|| **Indicates** | Tiles integration for Struts 2.x |
|| **Schema Entity** | `ViewRequirement.layout_framework = TILES` |

**Example web.xml:**
```xml
<listener>
    <listener-class>org.apache.struts2.tiles.StrutsTilesListener</listener-class>
</listener>
```

---

## Common Patterns

### Servlet Filter Configuration

#### Struts 1.x ActionServlet

|| Pattern | `org\.apache\.struts\.action\.ActionServlet` |
||---------|----------------------------------------------|
|| **File Types** | `web.xml` |
|| **Indicates** | Struts 1.x front controller |
|| **Schema Entity** | `ModernizationRequirements.source_technology = STRUTS_1` |

**Example:**
```xml
<servlet>
    <servlet-name>action</servlet-name>
    <servlet-class>org.apache.struts.action.ActionServlet</servlet-class>
    <init-param>
        <param-name>config</param-name>
        <param-value>/WEB-INF/struts-config.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>action</servlet-name>
    <url-pattern>*.do</url-pattern>
</servlet-mapping>
```

---

#### Struts 2.x Filter

|| Pattern | `org\.apache\.struts2\.dispatcher\.(FilterDispatcher\|StrutsPrepareAndExecuteFilter)` |
||---------|--------------------------------------------------------------------------------------|
|| **File Types** | `web.xml` |
|| **Indicates** | Struts 2.x front controller |
|| **Schema Entity** | `ModernizationRequirements.source_technology = STRUTS_2` |

**Example (modern Struts 2.x):**
```xml
<filter>
    <filter-name>struts2</filter-name>
    <filter-class>org.apache.struts2.dispatcher.filter.StrutsPrepareAndExecuteFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>struts2</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

**Example (older Struts 2.x):**
```xml
<filter>
    <filter-name>struts2</filter-name>
    <filter-class>org.apache.struts2.dispatcher.FilterDispatcher</filter-class>
</filter>
```

**Extraction Notes:**
- `FilterDispatcher` → deprecated in Struts 2.1+
- `StrutsPrepareAndExecuteFilter` → current recommended filter

---

### Properties Configuration

#### struts.properties

|| Pattern | `struts\.properties` (filename) |
||---------|--------------------------------|
|| **File Types** | `struts.properties` |
|| **Indicates** | Struts 2.x properties configuration |
|| **Schema Entity** | Configuration settings |

**Example:**
```properties
struts.devMode=false
struts.action.extension=action,do
struts.ui.theme=xhtml
struts.multipart.maxSize=2097152
struts.i18n.encoding=UTF-8
struts.objectFactory=spring
struts.convention.package.locators=action,actions
```

**Extraction Notes:**
- Alternative to `<constant>` in struts.xml
- Framework configuration settings
- Spring integration configuration

---

### File Upload Handling

#### Struts 1.x File Upload

|| Pattern | `org\.apache\.struts\.upload\.FormFile` |
||---------|----------------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | File upload in Struts 1.x |
|| **Schema Entity** | `FormField.type = FILE` |

**Example:**
```java
import org.apache.struts.upload.FormFile;

public class UploadForm extends ActionForm {
    private FormFile file;
    
    public FormFile getFile() { return file; }
    public void setFile(FormFile file) { this.file = file; }
}
```

---

#### Struts 2.x File Upload

|| Pattern | `private\s+File\s+\w+;\s*private\s+String\s+\w+ContentType;\s*private\s+String\s+\w+FileName;` |
||---------|------------------------------------------------------------------------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | File upload in Struts 2.x |
|| **Schema Entity** | `FormField.type = FILE` |

**Example:**
```java
public class UploadAction extends ActionSupport {
    private File upload;
    private String uploadContentType;
    private String uploadFileName;
    
    public String execute() {
        // Process uploaded file
        // 'upload' contains the temp file
        // 'uploadContentType' contains MIME type
        // 'uploadFileName' contains original filename
        return SUCCESS;
    }
    
    // Getters and setters
}
```

**Extraction Notes:**
- Struts 2.x creates three properties for each file input:
  - `{fieldName}` → File object
  - `{fieldName}ContentType` → MIME type
  - `{fieldName}FileName` → original filename
- Requires `fileUpload` interceptor (in defaultStack)

**Migration Path:**
- Spring Boot `MultipartFile` parameter
- `@RequestParam("file") MultipartFile file`

---

### Exception Handling

#### Struts 2.x Exception Mappings

|| Pattern | `<exception-mapping\s+exception=["'][^"']+["']\s+result=["'][^"']+["']` |
||---------|-------------------------------------------------------------------------|
|| **File Types** | `struts.xml` |
|| **Indicates** | Exception handling configuration |
|| **Schema Entity** | Error handling strategy |

**Example:**
```xml
<global-exception-mappings>
    <exception-mapping exception="java.lang.Exception" result="error"/>
    <exception-mapping exception="com.example.BusinessException" result="businessError"/>
</global-exception-mappings>

<action name="order">
    <exception-mapping exception="com.example.OrderNotFoundException" result="notFound"/>
    <result name="error">/WEB-INF/pages/error.jsp</result>
    <result name="notFound">/WEB-INF/pages/notfound.jsp</result>
</action>
```

---

## Migration Summary

### Modernization Considerations

When Struts (1.x or 2.x) is detected, add these to `ModernizationRequirements.migration_considerations`:

- **"Struts Framework Migration"** (high priority) → Core framework change
- **"View Technology Migration"** → JSP/Tiles to Thymeleaf
- **"Form Validation Migration"** → Struts validation to Bean Validation
- **"Interceptor Migration"** → Struts interceptors to Spring AOP/Filters
- **"Session State"** → If ActionForms use session scope or `@SessionAttributes`
- **"File Upload Migration"** → If file uploads detected
- **"OGNL to SpEL"** → Expression language change (Struts 2.x)

### Key Migration Mappings

|| Struts Concept | Spring Boot Equivalent |
||----------------|------------------------|
|| Action (1.x/2.x) | `@Controller` class |
|| Action method | `@GetMapping` / `@PostMapping` method |
|| ActionForm (1.x) | POJO / DTO with `@Valid` |
|| Action properties (2.x) | Method parameters (`@RequestParam`, `@ModelAttribute`) |
|| ActionForward (1.x) | Return view name or `RedirectView` |
|| Result code (2.x) | Return view name or redirect |
|| struts-config.xml / struts.xml | Java-based configuration (`@Configuration`) |
|| Form validation (XML) | Bean Validation annotations (`@NotNull`, `@Size`, etc.) |
|| `validate()` method | Bean Validation or custom validator |
|| Interceptor | Spring AOP `@Aspect` or servlet `Filter` |
|| Tiles | Thymeleaf Layout Dialect |
|| Struts tags | Thymeleaf attributes (`th:*`) |
|| OGNL expressions (2.x) | Thymeleaf expressions (`${...}`) |
|| `<bean:message>` / `<s:text>` | Thymeleaf `#{message.key}` |
|| ValueStack (2.x) | Spring `Model` / `@ModelAttribute` |
|| Request/session attributes | `Model`, `@SessionAttributes`, `HttpSession` |

### Example Migration

**Struts 1.x Action:**
```java
public class OrderAction extends Action {
    public ActionForward execute(ActionMapping mapping, ActionForm form,
            HttpServletRequest request, HttpServletResponse response) {
        OrderForm orderForm = (OrderForm) form;
        Order order = orderService.createOrder(orderForm);
        request.setAttribute("order", order);
        return mapping.findForward("success");
    }
}
```

**Spring Boot Controller:**
```java
@Controller
@RequestMapping("/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @PostMapping("/create")
    public String create(@Valid @ModelAttribute OrderDTO orderDTO, 
                        BindingResult result, Model model) {
        if (result.hasErrors()) {
            return "orders/form";
        }
        Order order = orderService.createOrder(orderDTO);
        model.addAttribute("order", order);
        return "redirect:/orders/list";
    }
}
```

---

**End of Struts Detection Patterns**

For UI layer patterns beyond Struts, see [ui-layer.md](ui-layer.md).
