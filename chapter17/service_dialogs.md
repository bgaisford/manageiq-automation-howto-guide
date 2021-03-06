## Service Dialogs

Service dialogs are used in several situations when working with CloudForms Automation, and we saw an example of creating a simple service dialog to use with a button in [A More Advanced Example](../chapter5/a_more_advanced_example.md). This example used two text boxes to prompt for simple text string values to pass to the Automation Method, but we can use several different element types when we create dialogs:

![screenshot](images/screenshot3.png)

The available element types are described in the CloudForms [Lifecycle and Automation Guide](https://access.redhat.com/documentation/en-US/Red_Hat_CloudForms/3.2/html-single/Lifecycle_and_Automation_Guide/index.html#sect-Service_Dialogs), although with ManageIQ _Botvinnik_ (CloudForms Management Engine 5.4) they gain several useful new features.

### Dynamic Elements

Prior to ManageIQ _Botvinnik_ only one element type was capable of dynamic (run-time) population - the **Dynamic Drop Down List**. Now with ManageIQ _Botvinnik_ most dialog element types are capable of dynamic population, and so the **Dynamic Drop Down List** has been removed as a separate element type.

Dynamic elements are populated from a Method, called either when the service dialog is initially displayed, or from an optional **Refresh** button. The URI to the Method is specified when we add the element and select the checkbox to make it dynamic.

#### Populating the Dynamic Fields

The dynamic element has/is its own `$evm.object`, and we need to populate some pre-defined hash key/value pairs to define the dialog field settings, and to load the data to be displayed. 

```ruby
dialog_field = $evm.object

# sort_by: value / description / none
dialog_field["sort_by"] = "value"

# sort_order: ascending / descending
dialog_field["sort_order"] = "ascending"

# data_type: string / integer
dialog_field["data_type"] = "integer"

# required: true / false
dialog_field["required"] = "true"

dialog_field["values"] = {1 => "one", 2 => "two", 10 => "ten", 50 => "fifty"}
dialog_field["default_value"] = 2
```

For a dynamic drop-down list, the `values` key of this hash is also a hash of key/value pairs, with each pair representing a value to be displayed in the element, and the corresponding 'data\_type' value to be returned to Automate as the **dialog\_*** option if that choice is selected.

Another more real-world example is:

```ruby
  values_hash = {}
  values_hash['!'] = '-- select from list --'
  user_group = $evm.root['user'].ldap_group
  #
  # Everyone can provision to DEV and UAT
  #
  values_hash['dev'] = "Development"
  values_hash['uat'] = "User Acceptance Test"
  if user_group.downcase =~ /administrators/
    #
    # Administrators can also provision to PRE-PROD and PROD
    #
    values_hash['pre-prod'] = "Pre-Production"
    values_hash['prod'] = "Production"
  end

  list_values = {
     'sort_by'    => :value,
     'data_type'  => :string,
     'required'   => true,
     'values'     => values_hash
  }
  list_values.each { |key, value| $evm.object[key] = value }
```

### Read-Only and Protected Elements

ManageIQ _Anand_ added the ability to be able to mark a text box as protected, which results in any input being obfuscated; useful for inputting passwords.

ManageIQ _Botvinnik_ introduced the concept of read-only elements for service dialogs, which cannot be changed once displayed. Having a text box dynamically populated, but read-only, makes it ideal for displaying messages.

<br>
![screenshot](images/screenshot5.png)

#### Programmatically Populating a Read-Only Text Box

We can use dynamically-populated read-only text or text area boxes as status boxes to display messages. Here is an example of populating a text box with a message, depending on whether the user is provisioning into Amazon or not:

```ruby
 if $evm.root['vm'].vendor.downcase == 'amazon' 
   status = "Valid for this VM type"
 else
   status = 'Invalid for this VM type'
 end
 list_values = {
    'required'   => true,
    'protected'   => false,
    'read_only'  => true,
    'value' => status,
  }
  list_values.each do |key, value| 
    $evm.object[key] = value
  end
```

### Element Validation

ManageIQ _Botvinnik_ introduced the ability to add input field validation to dialog elements. Currently the only Validator Types are **None** or  **Regular Expression**, but regular expressions are useful for validating input for values such as IP Addresses:

<br>
![screenshot](images/screenshot6.png)

### Using the Input from One Element in Another Element's Dynamic Method

We can link elements in such a way that a user's input in the one element can be used by subsequent dynamic elements that have the **Show Refresh Button** selected. The subsequent dynamic method, when refreshed, can access the first element's input value using `$evm.root['dialog_elementname']`.

We can use this in several useful ways, such as to populate a dynamic list based on a value input previously, or to create a validation method.

In the following example a read-only text area box element is used to display a validation message, with the user instructed to click the 'Refresh' button to validate their input to a previous field.

The service dialog has a text box element that prompts for the name of a new OpenStack tenant to be created in each of several OpenStack providers. The validation method checks that the tenant name doesn't already exist.

Until the **Refresh** button is clicked, the Validation text area box displays "Validation...". Once the **Refresh** button is clicked, the validation message changes according to whether the tenant exists or not.


```ruby
display_string = "Validation...\n"
tenant_found = false
#
# Read the input from the 'tenant' element
#
tenant_name = $evm.root['dialog_tenant_name']
$evm.log(:info, "Tenant name = \'#{tenant_name}\'")
unless tenant_name.length.zero?
  lowercase_tenant = tenant_name.gsub(/\W/,'_').downcase
  tenant_objects = $evm.vmdb('CloudTenant').find(:all)
  tenant_objects.each do | tenant |
    if tenant.name.downcase == lowercase_tenant
      tenant_found = true
      display_string += "   Tenant \'#{tenant.name}\' exists in OpenStack Provider: " 
      display_string += "#{$evm.vmdb('ems', tenant.ems_id).name}\n"
    end
  end
  display_string += "   Tenant \'#{lowercase_tenant}\' is available for use" unless tenant_found
end

list_values = {
  'required'   => true,
  'protected'   => false,
  'read_only'  => true,
  'value' => display_string,
}
list_values.each do |key, value| 
  $evm.log(:info, "Setting dialog variable #{key} to #{value}")
  $evm.object[key] = value
end
exit MIQ_OK
```

