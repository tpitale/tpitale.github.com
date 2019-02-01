---
published: false
title: Aliasing Fields on Ecto Associations in Absinthe for GraphQL
subtitle: 
author: Tony Pitale
created_at: 2019-02-01 11:10:24.786042 -06:00
layout: post
---

How do we maintain the consistency of our GraphQL APIs over time, while the underlying data structures change? A common question we face building software on the web. We have mobile and react applications, or third party consumers, that rely upon the structure and data provided our APIs. The truth of our data storage may be much different than the API presents. Let’s adapt a few common scenarios using features available from Absinthe in Elixir/Phoenix/Ecto.

One technique for handling this is to continue presenting the original field, but using data from another source. I would refer to this as “aliasing” or mapping one field’s data into another. The fundamental idea in Absinthe is that the name of the field presented in the API is not required to be the key for the source of data we use.

I’ll cover two scenarios of database refactorings I went through recently, while keeping the API backward compatible:

1. Renaming a column
2. Moving a column into another table

## Renaming a Column ##

This is a simple example to get us warmed up, but we’ll use this technique in combination with another in our next example.
We’ll start with a table and a column (example from Postgres):

```
CREATE TABLE firmware_versions (
  id SERIAL,
  semantic_version_number text
);

CREATE TABLE devices (
  id SERIAL,
  firmware_version_id integer REFERENCES firmware_versions(id),
  desired_firmware_version_id integer REFERENCES firmware_versions(id)
);
```

And the Absinthe object type would look something like (I’ve omitted data loaders and resolvers):

```
@desc "Firmware version"
object :firmware_version do
  @desc "Database ID of the firmware version"
  field(:id, :integer)

  @desc "Firmware version expressed as a semantic version number"
  field(:semantic_version_number, :string)
end

@desc "A network-connected device"
object :device do
  …
  @desc "Current firmware version of the device"
  field(:firmware_version, :firmware_version, resolve: dataloader(Device))

  @desc “Firmware version we are attempting to update to"
  field(:desired_firmware_version, :firmware_version, resolve: dataloader(Device))
end
```

What happens when we change the column name from `desired_firmware_version_id` to `requested_firmware_version_id`? An admittedly contrived example, meant to highlight a point. Aside from the normal database migration to alter the table, we need only make a straightforward change in our GraphQL type for the device to maintain the interface for our users.

```
field(:requested_firmware_version, :firmware_version, resolve: dataloader(Device), name: “desired_firmware_version”)
```

Two changes are required. Change the first argument in `field` to match the new name of the column (in this case, an association). And then, add the option for `name` and pass in a string (binary) to match the previous column’s name in the API.

We may also wish to make use of the `deprecate` option available to `field` and pass it a string with the reason for the deprecation.

## Moving to a New Table ##

Whether this is a new table or an existing table that better suits our data, our structures may change in more substantial ways. In this example, we’ll start from the same initial structure of the `devices` table from above. Instead, we’ll add a new table rather than renaming our column.

```
CREATE TABLE firmware_update_requests (
  id SERIAL,
  pending boolean DEFAULT true,
  device_id integer REFERENCES devices(id),
  firmware_version_id integer REFERENCES firmware_versions(id)
)

CREATE UNIQUE INDEX pending_request_index ON firmware_update_requests(device_id, pending);
```
This change will necessitate more changes than our previous example, but fundamentally, the technique is the same. To make it work, we just need to get the data from this new table’s `firmware_version_id` into a place on the device record where our alias `field` can get it.

To accomplish this, we’ll use a combination of two Ecto features. First, we’ll make space for the `firmware_version_id` (and association) using a `virtual` field in the schema.

```
# in MyApp.Device module
schema :devices do
  …
  field(:requested_firmware_version_id, :integer, virtual: true)
  belongs_to(:requested_firmware_version, MyApp.FirmwareVersion, define_field: false)
end
```

This creates a place for us to join data into the device, and allows us to look it up as an association. Now, we can change our query to load this data. For Absinthe, we do this in the resolver:

```
defp with_pending_request(query \\ MyApp.Device) do
  from(
    d in query,
    left_join: r in MyApp.FirmwareUpdateRequest,
    on: d.id == r.device_id,
    on: r.pending == true,
    select: %{d | requested_firmware_version_id: r.firmware_version_id}
  )
end
```

Let’s break this down. First thing is the `query` argument. This is an `Ecto.Query` for records from `MyApp.Device`. Next, we `left_join` our firmware update request record on the matching `device_id` and we restrict it to `pending` requests so that we only join at most a single record for each device.

Lastly, my favorite bit is the `select`. It merges the `firmware_version_id` into the new virtual field we added to device. And with that data in place, we can use our change to the Absinthe `object` for the `name` option and everything will work.

With all of these changes: loading the firmware version from the request association into a virtual field and aliasing the field in Absinthe we have successfully mapped a field from an associated table into place on the original field in our API.

While this is how I solved this particular problem it is, of course, not the only way to solve this problem. Other solutions might involve resolver callbacks coming in the next version of Absinthe, or simply a custom resolver that returns a properly formed map.