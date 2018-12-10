# Update key of all objects within PostgreSQLs jsonb array

Let's say we have a model that has a jsonb column with an array of objects like so:

```json
[
  {
    "hour": 16,
    "time": 0.1,
  }, {
    "hour": 18,
    "time": 0.2,
  }
]
```
and we want to add to every object additional key with some value. The `jsonb_set` function requires us to pass the index of each object, so we have to iterate over the collection. There is no built-in function, so we have to create a custom one. The implementation might look like that:

```SQL
-- the params are the same as in aforementioned `jsonb_set`
CREATE OR REPLACE FUNCTION update_array_elements(target jsonb, path text[], new_value jsonb)
  RETURNS jsonb language sql AS $$
  -- aggregate the jsonb from parts created in LATERAL
  SELECT jsonb_agg(updated_jsonb)
  -- split the target array to individual objects...
  FROM jsonb_array_elements(target) individual_object,
  -- operate on each object and apply jsonb_set to it. The results are aggregated in SELECT
  LATERAL jsonb_set(individual_object, path, new_value) updated_jsonb
$$;
```

And that's it... :)
