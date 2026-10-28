# UI Layer Patterns

These patterns detect presentation layer technologies that require special migration consideration since JSP, Tiles, Struts, and JSF are not directly supported in Spring Boot.

## Table of Contents

- [JSP Detection](#jsp-detection)
- [Apache Tiles Detection](#apache-tiles-detection)
- [Struts Detection](#struts-detection)
- [JSF Detection](#jsf-detection)
- [SiteMesh Detection](#sitemesh-detection)
- [Thymeleaf Detection](#thymeleaf-detection)
- [Freemarker Detection](#freemarker-detection)
- [Client-Side Assets Detection](#client-side-assets-detection)
- [Form Detection](#form-detection)
- [View Resolver Detection](#view-resolver-detection)
- [Model Attribute Detection](#model-attribute-detection)
- [Session Attributes Detection](#session-attributes-detection)
- [HTTP session usage (HttpSession, getSession, setAttribute, getAttribute)](#http-session-usage-httpsession-getsession-setattribute-getattribute)

---

## JSP Detection

### JSP Files

| Pattern | `\.jsp$` (filename pattern) |
|---------|------------------------------|
| **File Types** | `*.jsp` |
| **Indicates** | JavaServer Pages view technology |
| **Schema Entity** | `ViewRequirement.view_technology = JSP`, `ViewComponent` |

**Example:**
```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<html>
<head><title>Order List</title></head>
<body>
    <c:forEach items="${orders}" var="order">
        <p>${order.orderNumber}</p>
    </c:forEach>
</body>
</html>
```

**Extraction Notes:**
- Each JSP file becomes a `ViewComponent`
- Extract `<%@ taglib %>` directives for `TagLibrary` entries
- Extract EL expressions `${...}` for `ModelAttribute` identification
- Parse form elements for `FormDefinition`

---

### JSTL Core Tags

| Pattern | `<%@\s*taglib.*uri=["']http://java.sun.com/jsp/jstl/core["']` |
|---------|---------------------------------------------------------------|
| **File Types** | `*.jsp` |
| **Indicates** | JSTL core library usage |
| **Schema Entity** | `ViewRequirement.tag_libraries` |

**Example:**
```jsp
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<c:if test="${order.status == 'PENDING'}">
    <span class="pending">Pending</span>
</c:if>
<c:forEach items="${items}" var="item">
    ...
</c:forEach>
```

**Extraction Notes:**
- `<c:if>`, `<c:choose>` → Thymeleaf `th:if`, `th:switch`
- `<c:forEach>` → Thymeleaf `th:each`
- `<c:out>` → Thymeleaf `th:text`
- `<c:set>` → Thymeleaf `th:with`

---

### JSTL Formatting Tags

| Pattern | `<%@\s*taglib.*uri=["']http://java.sun.com/jsp/jstl/fmt["']` |
|---------|--------------------------------------------------------------|
| **File Types** | `*.jsp` |
| **Indicates** | JSTL formatting/i18n |
| **Schema Entity** | `ViewRequirement.tag_libraries`, `ViewComponent.i18n_keys` |

**Example:**
```jsp
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>
<fmt:message key="order.title"/>
<fmt:formatDate value="${order.createdDate}" pattern="yyyy-MM-dd"/>
<fmt:formatNumber value="${order.total}" type="currency"/>
```

**Extraction Notes:**
- Extract `<fmt:message key="...">` for i18n keys
- `<fmt:formatDate>` → Thymeleaf `#temporals` utility
- `<fmt:formatNumber>` → Thymeleaf `#numbers` utility

---

### Spring Form Tags

| Pattern | `<%@\s*taglib.*uri=["']http://www.springframework.org/tags/form["']` |
|---------|----------------------------------------------------------------------|
| **File Types** | `*.jsp` |
| **Indicates** | Spring MVC form binding |
| **Schema Entity** | `ViewRequirement.tag_libraries`, `FormDefinition` |

**Example:**
```jsp
<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>
<form:form modelAttribute="orderForm" action="/orders/create" method="POST">
    <form:input path="customerName" cssClass="form-control"/>
    <form:errors path="customerName" cssClass="error"/>
    <form:select path="status" items="${statusList}"/>
    <form:checkbox path="expedited" label="Expedited Shipping"/>
    <button type="submit">Submit</button>
</form:form>
```

**Extraction Notes:**
- Extract `modelAttribute` → `FormDefinition.model_object`
- Extract `path` attributes → `FormField.name`
- Extract `<form:errors>` → validation display requirements
- Maps to Thymeleaf `th:object` and `th:field`

---

### Spring Tags

| Pattern | `<%@\s*taglib.*uri=["']http://www.springframework.org/tags["']` |
|---------|----------------------------------------------------------------|
| **File Types** | `*.jsp` |
| **Indicates** | Spring JSP tags (messages, URLs, themes) |
| **Schema Entity** | `ViewRequirement.tag_libraries` |

**Example:**
```jsp
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags" %>
<spring:message code="welcome.message"/>
<spring:url value="/orders/{id}" var="orderUrl">
    <spring:param name="id" value="${order.id}"/>
</spring:url>
<a href="${orderUrl}">View Order</a>
```

---

### JSP Include Directive

| Pattern | `<%@\s*include\s+file=["'][^"']+["']` |
|---------|----------------------------------------|
| **File Types** | `*.jsp` |
| **Indicates** | Static JSP includes |
| **Schema Entity** | `ViewComponent.includes` |

**Example:**
```jsp
<%@ include file="/WEB-INF/views/common/header.jsp" %>
<div class="content">...</div>
<%@ include file="/WEB-INF/views/common/footer.jsp" %>
```

---

### JSP Dynamic Include

| Pattern | `<jsp:include\s+page=["'][^"']+["']` |
|---------|---------------------------------------|
| **File Types** | `*.jsp` |
| **Indicates** | Dynamic JSP includes |
| **Schema Entity** | `ViewComponent.includes` |

**Example:**
```jsp
<jsp:include page="/WEB-INF/views/orders/sidebar.jsp">
    <jsp:param name="activeTab" value="list"/>
</jsp:include>
```

---

## Apache Tiles Detection

### Tiles Configuration File

| Pattern | `tiles\.xml\|tiles-defs\.xml` (filename pattern) |
|---------|--------------------------------------------------|
| **File Types** | `*.xml` |
| **Indicates** | Apache Tiles framework |
| **Schema Entity** | `ViewRequirement.layout_framework = TILES`, `TilesDefinition` |

**Example:**
```xml
<!DOCTYPE tiles-definitions PUBLIC
    "-//Apache Software Foundation//DTD Tiles Configuration 3.0//EN"
    "http://tiles.apache.org/dtds/tiles-config_3_0.dtd">
<tiles-definitions>
    <definition name="base.layout" template="/WEB-INF/layouts/base.jsp">
        <put-attribute name="header" value="/WEB-INF/views/common/header.jsp"/>
        <put-attribute name="footer" value="/WEB-INF/views/common/footer.jsp"/>
    </definition>
    <definition name="orders.list" extends="base.layout">
        <put-attribute name="title" value="Order List"/>
        <put-attribute name="body" value="/WEB-INF/views/orders/list.jsp"/>
    </definition>
</tiles-definitions>
```

**Extraction Notes:**
- Each `<definition>` → `TilesDefinition`
- Extract `extends` for template inheritance
- Extract `<put-attribute>` → `TilesAttribute`
- Migration: Use Thymeleaf Layout Dialect

---

### Tiles Taglib

| Pattern | `<%@\s*taglib.*uri=["']http://tiles.apache.org/tags-tiles["']` |
|---------|----------------------------------------------------------------|
| **File Types** | `*.jsp` |
| **Indicates** | Tiles tag usage in JSP |
| **Schema Entity** | `ViewRequirement.tag_libraries` with `type = TILES` |

**Example:**
```jsp
<%@ taglib prefix="tiles" uri="http://tiles.apache.org/tags-tiles" %>
<tiles:insertAttribute name="header"/>
<div class="content">
    <tiles:insertAttribute name="body"/>
</div>
<tiles:insertAttribute name="footer"/>
```

---

### TilesConfigurer Bean

| Pattern | `TilesConfigurer\|TilesViewResolver` |
|---------|---------------------------------------|
| **File Types** | `*.xml`, `*.java` |
| **Indicates** | Spring Tiles integration |
| **Schema Entity** | `ViewRequirement.layout_framework = TILES` |

**Example:**
```xml
<bean id="tilesConfigurer" class="org.springframework.web.servlet.view.tiles3.TilesConfigurer">
    <property name="definitions">
        <list>
            <value>/WEB-INF/tiles/tiles-defs.xml</value>
        </list>
    </property>
</bean>
<bean id="viewResolver" class="org.springframework.web.servlet.view.tiles3.TilesViewResolver"/>
```

---
## Struts Detection

> **For comprehensive Struts patterns covering all versions (1.x, 2.x) and features, see [struts.md](struts.md).**

This section provides a quick reference for detecting Apache Struts applications. For detailed patterns including configuration files, action classes, form beans, validation, interceptors, Tiles integration, and complete migration guidance, refer to the dedicated Struts patterns document.

---

### Struts Version Detection

#### Struts 1.x Configuration File

|| Pattern | `struts-config\.xml` (filename pattern) |
||---------|----------------------------------------|
|| **File Types** | `struts-config.xml` |
|| **Indicates** | Struts 1.x framework |
|| **Schema Entity** | `ModernizationRequirements.source_technology = STRUTS_1` |

**Example:**
```xml
<struts-config>
    <form-beans>
        <form-bean name="orderForm" type="com.example.form.OrderForm"/>
    </form-beans>
    <action-mappings>
        <action path="/orders/create"
                type="com.example.action.CreateOrderAction"
                name="orderForm"
                validate="true"
                input="/WEB-INF/pages/orders/form.jsp">
            <forward name="success" path="/orders/list.do" redirect="true"/>
        </action>
    </action-mappings>
</struts-config>
```

**Migration:** See [struts.md](struts.md) for complete Struts 1.x patterns and Spring Boot migration.

---

#### Struts 2.x Configuration File

|| Pattern | `struts\.xml` (filename pattern) |
||---------|----------------------------------|
|| **File Types** | `struts.xml` |
|| **Indicates** | Struts 2.x framework |
|| **Schema Entity** | `ModernizationRequirements.source_technology = STRUTS_2` |

**Example:**
```xml
<struts>
    <package name="orders" extends="struts-default" namespace="/orders">
        <action name="list" class="com.example.action.OrderAction" method="list">
            <result>/WEB-INF/pages/orders/list.jsp</result>
        </action>
    </package>
</struts>
```

**Migration:** See [struts.md](struts.md) for complete Struts 2.x patterns and Spring Boot migration.

---

### Struts Action Classes (Quick Reference)

#### Struts 1.x Action

|| Pattern | `extends\s+Action\b` |
||---------|----------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Struts 1.x action implementation |
|| **Schema Entity** | `ServiceEndpoint`, `ServiceOperation` |

**Migration:** Convert to Spring `@Controller` with `@RequestMapping` methods.

See [struts.md - Struts 1.x Action Classes](struts.md#struts-1x-action-classes) for complete patterns including DispatchAction, LookupDispatchAction, and detailed extraction notes.

---

#### Struts 2.x Action

|| Pattern | `extends\s+ActionSupport\b\|implements\s+Action\b` |
||---------|-----------------------------------------------------|
|| **File Types** | `*.java` |
|| **Indicates** | Struts 2.x action implementation |
|| **Schema Entity** | `ServiceEndpoint`, `ServiceOperation` |

**Migration:** Convert to Spring `@Controller` with HTTP method annotations.

See [struts.md - Struts 2.x Action Classes](struts.md#struts-2x-action-classes) for complete patterns including POJO actions, annotations, and detailed extraction notes.

---

### Struts Tag Libraries (Quick Reference)

#### Struts 1.x Tags

|| Pattern | `<%@\s*taglib.*uri=["']/tags/struts-(html\|bean\|logic)["']` |
||---------|--------------------------------------------------------------|
|| **File Types** | `*.jsp` |
|| **Indicates** | Struts 1.x tag library usage |
|| **Schema Entity** | `ViewRequirement.tag_libraries` with `type = STRUTS_1` |

**Common prefixes:** `html`, `bean`, `logic`

**Migration:** Convert to Thymeleaf attributes.

See [struts.md - Struts 1.x Tag Libraries](struts.md#struts-1x-tag-libraries) for complete tag patterns and Thymeleaf migration mappings.

---

#### Struts 2.x Tags

|| Pattern | `<%@\s*taglib.*uri=["']/struts-tags["']` |
||---------|------------------------------------------|
|| **File Types** | `*.jsp` |
|| **Indicates** | Struts 2.x tag library usage |
|| **Schema Entity** | `ViewRequirement.tag_libraries` with `type = STRUTS_2` |

**Common prefix:** `s`

**Migration:** Convert to Thymeleaf attributes.

See [struts.md - Struts 2.x Tag Libraries](struts.md#struts-2x-tag-libraries) for complete tag patterns, OGNL expressions, and Thymeleaf migration mappings.

---

### Additional Struts Patterns

For comprehensive coverage including:

- **Configuration:** Form beans, action mappings, validation XML, struts.properties
- **Validation:** XML-based validation, validation annotations, custom validators
- **Interceptors:** Custom interceptors, interceptor stacks, built-in interceptors
- **Tiles Integration:** Tiles 1.x, 2.x, 3.x configuration and usage
- **Annotations:** @Action, @Result, @Namespace, validation annotations
- **Convention Plugin:** Package-based action discovery
- **File Uploads:** Struts 1.x FormFile and Struts 2.x file handling
- **Exception Handling:** Exception mappings and error handling
- **Complete Migration Guide:** Detailed Spring Boot equivalents for all Struts constructs

**See [struts.md](struts.md) for the complete Struts detection patterns documentation.**

---


## JSF Detection

### faces-config.xml

| Pattern | `faces-config\.xml` (filename pattern) |
|---------|----------------------------------------|
| **File Types** | `faces-config.xml` |
| **Indicates** | JavaServer Faces framework |
| **Schema Entity** | `ModernizationRequirements.source_technology = JSF`, `JSFManagedBean` |

**Example:**
```xml
<faces-config xmlns="http://xmlns.jcp.org/xml/ns/javaee">
    <managed-bean>
        <managed-bean-name>orderBean</managed-bean-name>
        <managed-bean-class>com.example.web.OrderBean</managed-bean-class>
        <managed-bean-scope>view</managed-bean-scope>
        <managed-property>
            <property-name>orderService</property-name>
            <value>#{orderService}</value>
        </managed-property>
    </managed-bean>
    <navigation-rule>
        <from-view-id>/orders/form.xhtml</from-view-id>
        <navigation-case>
            <from-outcome>success</from-outcome>
            <to-view-id>/orders/list.xhtml</to-view-id>
            <redirect/>
        </navigation-case>
    </navigation-rule>
</faces-config>
```

**Extraction Notes:**
- `<managed-bean>` → `JSFManagedBean`
- `<navigation-rule>` → `NavigationFlow`
- Migration: Convert to `@Service` + `@Controller`

---

### JSF Annotations

| Pattern | `@ManagedBean\|@Named\|@ViewScoped\|@SessionScoped\|@RequestScoped` |
|---------|---------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | JSF/CDI managed bean |
| **Schema Entity** | `JSFManagedBean` |

**Example:**
```java
@Named("orderBean")
@ViewScoped
public class OrderBean implements Serializable {
    
    @Inject
    private OrderService orderService;
    
    private Order order;
    
    public String createOrder() {
        orderService.save(order);
        return "success";
    }
    
    public void onProductSelect(AjaxBehaviorEvent event) {
        // AJAX handler
    }
}
```

**Extraction Notes:**
- Extract scope annotation → `JSFManagedBean.scope`
- Methods returning String → action methods (navigation outcomes)
- Methods with event parameters → AJAX listeners

---

### Facelets Files

| Pattern | `\.xhtml$` (filename pattern with JSF namespace) |
|---------|--------------------------------------------------|
| **File Types** | `*.xhtml` |
| **Indicates** | Facelets view technology |
| **Schema Entity** | `ViewRequirement.view_technology = FACELETS`, `ViewComponent` |

**Example:**
```xhtml
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:h="http://xmlns.jcp.org/jsf/html"
      xmlns:f="http://xmlns.jcp.org/jsf/core"
      xmlns:ui="http://xmlns.jcp.org/jsf/facelets">
<h:head>
    <title>Order Form</title>
</h:head>
<h:body>
    <h:form id="orderForm">
        <h:inputText value="#{orderBean.order.customerName}">
            <f:validateRequired/>
        </h:inputText>
        <h:message for="orderForm:customerName"/>
        <h:commandButton value="Submit" action="#{orderBean.createOrder}">
            <f:ajax execute="@form" render="@form"/>
        </h:commandButton>
    </h:form>
</h:body>
</html>
```

**Extraction Notes:**
- `<h:form>` → `FormDefinition`
- `<h:inputText>`, `<h:selectOneMenu>` → `FormField`
- `<f:validate*>` → `FieldValidation`
- `<f:ajax>` → AJAX requirements
- `#{bean.method}` → action methods

---

### JSF Composite Components

| Pattern | `xmlns:cc=["']http://xmlns.jcp.org/jsf/composite` |
|---------|---------------------------------------------------|
| **File Types** | `*.xhtml` |
| **Indicates** | JSF composite components |
| **Schema Entity** | `ViewComponent` with `type = FRAGMENT` |

---

### PrimeFaces Detection

| Pattern | `xmlns:p=["']http://primefaces.org` |
|---------|--------------------------------------|
| **File Types** | `*.xhtml` |
| **Indicates** | PrimeFaces component library |
| **Schema Entity** | `ViewRequirement.tag_libraries` |

**Example:**
```xhtml
<html xmlns:p="http://primefaces.org/ui">
    <p:dataTable value="#{orderBean.orders}" var="order">
        <p:column headerText="Order #">
            <h:outputText value="#{order.orderNumber}"/>
        </p:column>
    </p:dataTable>
    <p:dialog widgetVar="orderDialog">
        ...
    </p:dialog>
</html>
```

**Extraction Notes:**
- PrimeFaces components have rich functionality
- Document specific components for migration planning
- Consider modern alternatives (DataTables, React components)

---

## SiteMesh Detection

### SiteMesh Configuration

| Pattern | `sitemesh\.xml\|decorators\.xml` (filename pattern) |
|---------|-----------------------------------------------------|
| **File Types** | `*.xml` |
| **Indicates** | SiteMesh layout framework |
| **Schema Entity** | `ViewRequirement.layout_framework = SITEMESH` |

**Example:**
```xml
<sitemesh>
    <mapping path="/*" decorator="/WEB-INF/decorators/main.jsp"/>
    <mapping path="/admin/*" decorator="/WEB-INF/decorators/admin.jsp"/>
    <excludes>
        <pattern>/api/*</pattern>
    </excludes>
</sitemesh>
```

**Extraction Notes:**
- Migration: Use Thymeleaf Layout Dialect
- Decorator patterns → Thymeleaf fragments

---

### SiteMesh Decorator Tags

| Pattern | `<decorator:` |
|---------|---------------|
| **File Types** | `*.jsp` |
| **Indicates** | SiteMesh decorator usage |
| **Schema Entity** | `ViewRequirement.layout_framework = SITEMESH` |

**Example:**
```jsp
<%@ taglib prefix="decorator" uri="http://www.opensymphony.com/sitemesh/decorator" %>
<html>
<head>
    <title><decorator:title default="My App"/></title>
    <decorator:head/>
</head>
<body>
    <div class="header">...</div>
    <div class="content">
        <decorator:body/>
    </div>
    <div class="footer">...</div>
</body>
</html>
```

---

## Thymeleaf Detection

### Thymeleaf Template Files

| Pattern | `\.html$` with `th:` attributes |
|---------|----------------------------------|
| **File Types** | `*.html` |
| **Indicates** | Thymeleaf view technology |
| **Schema Entity** | `ViewRequirement.view_technology = THYMELEAF`, `ViewComponent` |

**Example:**
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title th:text="${pageTitle}">Default Title</title>
</head>
<body>
    <h1 th:text="${order.orderNumber}">Order #</h1>
    <p th:text="${order.status}">Status</p>
</body>
</html>
```

**Extraction Notes:**
- Thymeleaf is the recommended view technology for Spring Boot
- Files typically in `src/main/resources/templates/`
- Natural templating: valid HTML that renders in browsers without processing

---

### Thymeleaf Namespace Declaration

| Pattern | `xmlns:th=["']http://www.thymeleaf.org["']` |
|---------|---------------------------------------------|
| **File Types** | `*.html` |
| **Indicates** | Thymeleaf template |
| **Schema Entity** | `ViewRequirement.view_technology = THYMELEAF` |

---

### Thymeleaf Text and Variable Expressions

| Pattern | `th:text=\|th:utext=\|\$\{[^}]+\}` |
|---------|-----------------------------------|
| **File Types** | `*.html` |
| **Indicates** | Thymeleaf variable expressions |
| **Schema Entity** | `ViewComponent.model_attributes` |

**Example:**
```html
<!-- Escaped text output -->
<span th:text="${user.name}">John Doe</span>

<!-- Unescaped HTML output -->
<div th:utext="${article.content}">Article content here</div>

<!-- Inline expressions -->
<p>Welcome, [[${user.name}]]!</p>
```

**Extraction Notes:**
- `th:text` escapes HTML (safe for user input)
- `th:utext` renders raw HTML (use with caution)
- `[[...]]` and `[(...)]]` for inline expressions

---

### Thymeleaf Iteration

| Pattern | `th:each=["'][^"']+["']` |
|---------|--------------------------|
| **File Types** | `*.html` |
| **Indicates** | Collection iteration |
| **Schema Entity** | `ViewComponent.model_attributes` |

**Example:**
```html
<table>
    <tr th:each="order : ${orders}">
        <td th:text="${order.orderNumber}">ORD-001</td>
        <td th:text="${order.status}">PENDING</td>
        <td th:text="${order.total}">$0.00</td>
    </tr>
</table>

<!-- With iteration status -->
<ul>
    <li th:each="item, iterStat : ${items}" 
        th:class="${iterStat.odd} ? 'odd' : 'even'">
        <span th:text="${iterStat.index}">0</span>: 
        <span th:text="${item.name}">Item</span>
    </li>
</ul>
```

**Extraction Notes:**
- `th:each` replaces JSTL `<c:forEach>`
- Iteration status variable provides `index`, `count`, `size`, `odd`, `even`, `first`, `last`

---

### Thymeleaf Conditionals

| Pattern | `th:if=\|th:unless=\|th:switch=\|th:case=` |
|---------|-------------------------------------------|
| **File Types** | `*.html` |
| **Indicates** | Conditional rendering |
| **Schema Entity** | `ViewComponent.conditional_display` |

**Example:**
```html
<!-- Simple conditional -->
<button th:if="${order.status == 'PENDING'}" type="submit">Cancel Order</button>

<!-- Negative conditional -->
<span th:unless="${user.active}">Account Inactive</span>

<!-- Switch statement -->
<div th:switch="${order.status}">
    <p th:case="'PENDING'">Order is pending</p>
    <p th:case="'SHIPPED'">Order has shipped</p>
    <p th:case="*">Unknown status</p>
</div>
```

**Extraction Notes:**
- `th:if` replaces JSTL `<c:if>`
- `th:switch`/`th:case` replaces JSTL `<c:choose>`/`<c:when>`
- `th:case="*"` is the default case

---

### Thymeleaf Form Binding

| Pattern | `th:object=\|th:field=\|th:action=` |
|---------|-------------------------------------|
| **File Types** | `*.html` |
| **Indicates** | Form binding to model objects |
| **Schema Entity** | `FormDefinition` |

**Example:**
```html
<form th:action="@{/orders/create}" th:object="${orderForm}" method="post">
    <input type="text" th:field="*{customerName}" />
    <span th:if="${#fields.hasErrors('customerName')}" 
          th:errors="*{customerName}" class="error">Error</span>
    
    <select th:field="*{status}">
        <option th:each="s : ${statusList}" 
                th:value="${s}" 
                th:text="${s.displayName}">Status</option>
    </select>
    
    <button type="submit">Submit</button>
</form>
```

**Extraction Notes:**
- `th:object` binds form to model attribute (like Spring's `modelAttribute`)
- `th:field="*{...}"` uses selection expression for field binding
- `th:errors` displays validation errors
- `@{...}` is URL expression (handles context path)

---

### Thymeleaf Fragments and Layouts

| Pattern | `th:fragment=\|th:insert=\|th:replace=\|th:include=` |
|---------|-----------------------------------------------------|
| **File Types** | `*.html` |
| **Indicates** | Template composition |
| **Schema Entity** | `ViewComponent.includes` |

**Example:**
```html
<!-- Fragment definition (fragments/header.html) -->
<header th:fragment="header(title)">
    <h1 th:text="${title}">Page Title</h1>
    <nav>...</nav>
</header>

<!-- Fragment usage -->
<div th:insert="~{fragments/header :: header('Order List')}"></div>
<div th:replace="~{fragments/footer :: footer}"></div>
```

**Extraction Notes:**
- `th:insert` inserts fragment inside the host tag
- `th:replace` replaces the host tag with the fragment
- `~{template :: fragment}` is fragment expression syntax
- Used for layouts, headers, footers, and reusable components

---

### Thymeleaf Layout Dialect

| Pattern | `layout:decorate=\|layout:fragment=` |
|---------|--------------------------------------|
| **File Types** | `*.html` |
| **Indicates** | Thymeleaf Layout Dialect (for page layouts) |
| **Schema Entity** | `ViewRequirement.layout_framework = THYMELEAF_LAYOUT` |

**Example:**
```html
<!-- Layout template (layouts/base.html) -->
<!DOCTYPE html>
<html xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout">
<head>
    <title layout:title-pattern="$CONTENT_TITLE - $LAYOUT_TITLE">My App</title>
</head>
<body>
    <header>...</header>
    <main layout:fragment="content">Default content</main>
    <footer>...</footer>
</body>
</html>

<!-- Page using layout -->
<!DOCTYPE html>
<html xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
      layout:decorate="~{layouts/base}">
<head>
    <title>Order List</title>
</head>
<body>
    <main layout:fragment="content">
        <h1>Orders</h1>
        <!-- Page-specific content -->
    </main>
</body>
</html>
```

**Extraction Notes:**
- Thymeleaf Layout Dialect provides Tiles-like layout functionality
- `layout:decorate` specifies the parent layout
- `layout:fragment` defines content sections
- Migration target for Apache Tiles applications

---

### ThymeleafViewResolver Configuration

| Pattern | `ThymeleafViewResolver\|SpringTemplateEngine\|SpringResourceTemplateResolver` |
|---------|-------------------------------------------------------------------------------|
| **File Types** | `*.xml`, `*.java` |
| **Indicates** | Thymeleaf Spring integration |
| **Schema Entity** | `ViewRequirement.view_technology = THYMELEAF` |

**Example (Java Config):**
```java
@Bean
public SpringResourceTemplateResolver templateResolver() {
    SpringResourceTemplateResolver resolver = new SpringResourceTemplateResolver();
    resolver.setPrefix("classpath:/templates/");
    resolver.setSuffix(".html");
    resolver.setTemplateMode(TemplateMode.HTML);
    return resolver;
}

@Bean
public SpringTemplateEngine templateEngine() {
    SpringTemplateEngine engine = new SpringTemplateEngine();
    engine.setTemplateResolver(templateResolver());
    engine.setEnableSpringELCompiler(true);
    return engine;
}

@Bean
public ThymeleafViewResolver viewResolver() {
    ThymeleafViewResolver resolver = new ThymeleafViewResolver();
    resolver.setTemplateEngine(templateEngine());
    return resolver;
}
```

**Extraction Notes:**
- Extract `prefix` and `suffix` for template locations
- Spring Boot auto-configures Thymeleaf when on classpath

---

## Freemarker Detection

### Freemarker Template Files

| Pattern | `\.ftl$\|\.ftlh$` (filename pattern) |
|---------|--------------------------------------|
| **File Types** | `*.ftl`, `*.ftlh` |
| **Indicates** | Freemarker view technology |
| **Schema Entity** | `ViewRequirement.view_technology = FREEMARKER`, `ViewComponent` |

**Example:**
```ftl
<!DOCTYPE html>
<html>
<head>
    <title>${pageTitle}</title>
</head>
<body>
    <h1>Order: ${order.orderNumber}</h1>
    <p>Status: ${order.status}</p>
</body>
</html>
```

**Extraction Notes:**
- `.ftl` is the standard extension
- `.ftlh` is for HTML-specific templates (auto-escaping enabled)
- Files typically in `src/main/resources/templates/`

---

### Freemarker Variable Interpolation

| Pattern | `\$\{[^}]+\}` |
|---------|---------------|
| **File Types** | `*.ftl`, `*.ftlh` |
| **Indicates** | Freemarker variable expressions |
| **Schema Entity** | `ViewComponent.model_attributes` |

**Example:**
```ftl
<!-- Simple variable -->
<span>${user.name}</span>

<!-- With default value -->
<span>${user.nickname!"Guest"}</span>

<!-- Null-safe navigation -->
<span>${user.address.city!""}</span>

<!-- Built-in formatting -->
<span>${order.total?string.currency}</span>
<span>${order.createdDate?string("yyyy-MM-dd")}</span>
```

**Extraction Notes:**
- `!` operator provides default values
- `?` accesses built-in functions
- Similar to EL expressions but with different syntax

---

### Freemarker Directives

| Pattern | `<#list\|<#if\|<#else\|<#switch\|<#include\|<#import\|<#macro` |
|---------|---------------------------------------------------------------|
| **File Types** | `*.ftl`, `*.ftlh` |
| **Indicates** | Freemarker control structures |
| **Schema Entity** | `ViewComponent` |

**Example:**
```ftl
<!-- Iteration -->
<#list orders as order>
    <tr>
        <td>${order.orderNumber}</td>
        <td>${order.status}</td>
    </tr>
</#list>

<!-- Conditional -->
<#if order.status == "PENDING">
    <button>Cancel Order</button>
<#elseif order.status == "SHIPPED">
    <span>In Transit</span>
<#else>
    <span>Completed</span>
</#if>

<!-- Switch -->
<#switch order.status>
    <#case "PENDING">Pending<#break>
    <#case "SHIPPED">Shipped<#break>
    <#default>Unknown
</#switch>
```

**Extraction Notes:**
- `<#list>` replaces JSTL `<c:forEach>`
- `<#if>` replaces JSTL `<c:if>` and `<c:choose>`
- Directives use `<#...>` syntax

---

### Freemarker Macros and Includes

| Pattern | `<#macro\|<#include\|<#import` |
|---------|-------------------------------|
| **File Types** | `*.ftl`, `*.ftlh` |
| **Indicates** | Template composition |
| **Schema Entity** | `ViewComponent.includes` |

**Example:**
```ftl
<!-- Macro definition -->
<#macro orderRow order>
    <tr>
        <td>${order.orderNumber}</td>
        <td>${order.status}</td>
    </tr>
</#macro>

<!-- Macro usage -->
<#list orders as order>
    <@orderRow order=order />
</#list>

<!-- Include another template -->
<#include "common/header.ftl">

<!-- Import with namespace -->
<#import "macros/forms.ftl" as forms>
<@forms.input name="customerName" label="Customer Name" />
```

**Extraction Notes:**
- Macros are reusable template components
- `<#include>` is static inclusion
- `<#import>` creates a namespace for macro libraries

---

### Freemarker Spring Form Macros

| Pattern | `<@spring\.\|spring\.ftl` |
|---------|---------------------------|
| **File Types** | `*.ftl`, `*.ftlh` |
| **Indicates** | Spring MVC form binding with Freemarker |
| **Schema Entity** | `FormDefinition` |

**Example:**
```ftl
<#import "spring.ftl" as spring>

<form action="/orders/create" method="post">
    <@spring.formInput "orderForm.customerName" />
    <@spring.showErrors "<br>" />
    
    <@spring.formSingleSelect "orderForm.status" statusOptions />
    
    <@spring.formTextarea "orderForm.notes" />
    
    <button type="submit">Submit</button>
</form>
```

**Extraction Notes:**
- Spring provides `spring.ftl` macro library
- `@spring.formInput` generates input with binding
- `@spring.showErrors` displays validation errors
- Similar functionality to Spring JSP form tags

---

### FreeMarkerViewResolver Configuration

| Pattern | `FreeMarkerViewResolver\|FreeMarkerConfigurer` |
|---------|------------------------------------------------|
| **File Types** | `*.xml`, `*.java` |
| **Indicates** | Freemarker Spring integration |
| **Schema Entity** | `ViewRequirement.view_technology = FREEMARKER` |

**Example (Java Config):**
```java
@Bean
public FreeMarkerConfigurer freeMarkerConfigurer() {
    FreeMarkerConfigurer configurer = new FreeMarkerConfigurer();
    configurer.setTemplateLoaderPath("classpath:/templates/");
    configurer.setDefaultEncoding("UTF-8");
    return configurer;
}

@Bean
public FreeMarkerViewResolver freeMarkerViewResolver() {
    FreeMarkerViewResolver resolver = new FreeMarkerViewResolver();
    resolver.setCache(true);
    resolver.setPrefix("");
    resolver.setSuffix(".ftl");
    resolver.setContentType("text/html;charset=UTF-8");
    return resolver;
}
```

**Extraction Notes:**
- Extract `templateLoaderPath` for template locations
- Extract `suffix` for file extension
- Spring Boot auto-configures Freemarker when on classpath

---

## Client-Side Assets Detection

### JavaScript Files in Views

| Pattern | `<script\s+src=["'][^"']+\.js["']` |
|---------|-------------------------------------|
| **File Types** | `*.jsp`, `*.html`, `*.xhtml` |
| **Indicates** | JavaScript dependencies |
| **Schema Entity** | `ViewComponent.javascript_files` |

**Example:**
```html
<script src="/js/jquery.min.js"></script>
<script src="/js/orders.js"></script>
<script src="/js/validation.js"></script>
```

**Extraction Notes:**
- Identify JavaScript frameworks (jQuery, etc.)
- Extract custom JS files for migration
- Consider bundling/modern build tools

---

### Inline JavaScript with AJAX

| Pattern | `\$\.ajax\|XMLHttpRequest\|fetch\s*\(` |
|---------|----------------------------------------|
| **File Types** | `*.js`, `*.jsp`, `*.html` |
| **Indicates** | AJAX calls |
| **Schema Entity** | `ViewComponent.ajax_endpoints` |

**Example:**
```javascript
$.ajax({
    url: '/api/orders/' + orderId + '/status',
    method: 'GET',
    success: function(data) {
        $('#status').text(data.status);
    }
});
```

**Extraction Notes:**
- Extract AJAX endpoints for API documentation
- These become REST API requirements
- Ensure endpoints are included in `ServiceEndpoint`

---

### CSS Files

| Pattern | `<link.*rel=["']stylesheet["'].*href=["'][^"']+\.css["']` |
|---------|-----------------------------------------------------------|
| **File Types** | `*.jsp`, `*.html`, `*.xhtml` |
| **Indicates** | CSS dependencies |
| **Schema Entity** | `ViewComponent.css_files` |

---

## Form Detection

### HTML Form Elements

| Pattern | `<form\s+[^>]*action=["'][^"']+["']` |
|---------|---------------------------------------|
| **File Types** | `*.jsp`, `*.html`, `*.xhtml` |
| **Indicates** | Form submission |
| **Schema Entity** | `FormDefinition` |

**Example:**
```html
<form action="/orders/create" method="POST" enctype="multipart/form-data">
    <input type="text" name="customerName" required maxlength="100"/>
    <input type="email" name="email" required/>
    <select name="productId">
        <option value="1">Product A</option>
    </select>
    <input type="file" name="attachment"/>
    <button type="submit">Submit</button>
</form>
```

**Extraction Notes:**
- Extract `action` → `FormDefinition.action`
- Extract `method` → `FormDefinition.method`
- Extract `enctype` for file upload detection
- Extract input elements → `FormField`
- Extract validation attributes (`required`, `maxlength`, `pattern`)

---

### Client-Side Validation

| Pattern | `required\|pattern=["'][^"']+["']\|maxlength=["']\d+["']` |
|---------|-----------------------------------------------------------|
| **File Types** | `*.jsp`, `*.html`, `*.xhtml` |
| **Indicates** | HTML5 validation attributes |
| **Schema Entity** | `FormField.validations` |

---

### jQuery Validation Plugin

| Pattern | `\.validate\s*\(\|rules:\s*\{` |
|---------|--------------------------------|
| **File Types** | `*.js` |
| **Indicates** | jQuery Validation plugin |
| **Schema Entity** | `FormDefinition.validation_type = CLIENT_SIDE` or `BOTH` |

**Example:**
```javascript
$("#orderForm").validate({
    rules: {
        customerName: {
            required: true,
            maxlength: 100
        },
        email: {
            required: true,
            email: true
        }
    },
    messages: {
        customerName: {
            required: "Customer name is required"
        }
    }
});
```

---

## View Resolver Detection

### InternalResourceViewResolver

| Pattern | `InternalResourceViewResolver` |
|---------|--------------------------------|
| **File Types** | `*.xml`, `*.java` |
| **Indicates** | JSP view resolution |
| **Schema Entity** | `ViewRequirement.view_resolver_config` |

**Example:**
```xml
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/views/"/>
    <property name="suffix" value=".jsp"/>
    <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
</bean>
```

**Extraction Notes:**
- Extract `prefix` and `suffix` for template locations
- Extract `viewClass` for view technology identification

---

### Multiple View Resolvers

| Pattern | `ContentNegotiatingViewResolver\|BeanNameViewResolver` |
|---------|--------------------------------------------------------|
| **File Types** | `*.xml`, `*.java` |
| **Indicates** | Multiple view technologies |
| **Schema Entity** | `ViewRequirement` (may need multiple entries) |

---

## Model Attribute Detection

### @ModelAttribute Annotation

| Pattern | `@ModelAttribute\s*\(["'][^"']+["']\)` |
|---------|----------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Model data binding |
| **Schema Entity** | `ViewComponent.model_attributes` |

**Example:**
```java
@Controller
public class OrderController {
    
    @ModelAttribute("statusList")
    public List<OrderStatus> getStatusList() {
        return Arrays.asList(OrderStatus.values());
    }
    
    @GetMapping("/orders/form")
    public String showForm(@ModelAttribute("order") OrderDTO order) {
        return "orders/form";
    }
}
```

**Extraction Notes:**
- `@ModelAttribute` on methods → shared model data
- `@ModelAttribute` on parameters → form backing objects
- Extract for `ViewComponent.model_attributes`

---

## Session Attributes Detection

### @SessionAttributes Annotation

| Pattern | `@SessionAttributes` |
|---------|----------------------|
| **File Types** | `*.java` |
| **Indicates** | Session-scoped model attributes |
| **Schema Entity** | `NavigationFlow.state_management = SESSION`; also `ModernizationRequirements.migration_considerations` → include `"Session State"` |

**Example:**
```java
@Controller
@SessionAttributes("checkoutCart")
public class CheckoutController {
    
    @ModelAttribute("checkoutCart")
    public Cart initCart() {
        return new Cart();
    }
    
    @PostMapping("/checkout/complete")
    public String complete(SessionStatus status) {
        status.setComplete(); // Clear session attributes
        return "redirect:/orders/confirmation";
    }
}
```

**Extraction Notes:**
- Indicates stateful controller behavior
- Document for migration to stateless design
- Add `"Session State"` to `ModernizationRequirements.migration_considerations`
- Consider alternatives: JWT tokens, client-side state

### HTTP session usage (HttpSession, getSession, setAttribute, getAttribute)

| Pattern | `HttpSession` \| `getSession()` \| `setAttribute(` \| `getAttribute(` |
|---------|----------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Application stores or retrieves data in the HTTP session |
| **Schema Entity** | `ModernizationRequirements.migration_considerations` → include `"Session State"`; optionally `NavigationFlow.state_management = SESSION` when flow state is in session |

**Example:**
```java
@GetMapping("/cart")
public String viewCart(HttpServletRequest request) {
    HttpSession session = request.getSession();
    Cart cart = (Cart) session.getAttribute("cart");
    if (cart == null) {
        cart = new Cart();
        session.setAttribute("cart", cart);
    }
    return "cart";
}
```

**Extraction Notes:**
- Any use of `HttpSession`, `request.getSession()`, `session.setAttribute(`, or `session.getAttribute(` indicates session-stored state
- Add `"Session State"` to `ModernizationRequirements.migration_considerations`
- JSF `@SessionScoped` (see [JSF detection](#jsf-detection)) also indicates session state; include `"Session State"` when present
