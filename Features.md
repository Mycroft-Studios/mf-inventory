## Item degradation
Items automatically degrade over time (if the config option is enabled).
If the item reaches 0 quality, if possible, the item will degrade into its counterpart.
To create a degraded counterpart for an item, simply add it into the database under the same name as your item,
prefixed by "degraded_".

## Unique
The unique column added to your database determines whether or not an item is allowed to be stacked.
Items such as weapons and containers are unique by default (enforced through code).

## DegradeModifier
The degrademodifier column that is added to your items database table through the sql insert query is responsible for varying
the time it takes to degrade different types of items. 0.0 = no degradation, 1.0 = default degradation, 2.0+ = faster degradation.

## Container Items
Container items are items that are capable of being used to open another sub-inventory (e.g: tacobell food bag).
You can define more container items in the config.lua.