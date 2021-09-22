# Documentation
[View the repository](https://github.com/overextended/oxmysql/)

## Installation
* Download `oxmysql.zip` from https://github.com/overextended/oxmysql/releases/latest
* Extract `oxmysql` to your resources folder
* Add `ensure oxmysql` before your resources in `server.cfg`

## Configuration
```
set mysql_connection_string "mysql://user:password@host/database?charset=utf8mb4"
set mysql_slow_query_warning 100
```

## Prepared statements
When a query has been prepared it is stored into cache for faster execution in the future, therefore it is good practice to utilise parameters to reduce the number of prepared statements.  
Unless it is your only option you should avoid string concatenation in your queries, and never use it for queries executed from client-input.
```lua
-- These four queries will use two prepared statements (one for group, one for job), only requiring the parameters to be passed
local id1, id2 = 'abcd', 'efgh'
local result = exports.oxmysql:scalarSync('SELECT group FROM users WHERE identifier = ?', {id1})
local result = exports.oxmysql:scalarSync('SELECT group FROM users WHERE identifier = ?', {id2})
local result = exports.oxmysql:scalarSync('SELECT job FROM users WHERE identifier = ?', {id1})
local result = exports.oxmysql:scalarSync('SELECT job FROM users WHERE identifier = ?', {id2})

-- Each one of these queries is stored separately and will only improve the speed when called exactly as they are stored
local id1, id2 = 'abcd', 'efgh'
local result = exports.oxmysql:scalarSync('SELECT group FROM users WHERE identifier = "'..id1..'"')
local result = exports.oxmysql:scalarSync('SELECT group FROM users WHERE identifier = "'..id2..'"')
local result = exports.oxmysql:scalarSync('SELECT job FROM users WHERE identifier = "'..id1..'"')
local result = exports.oxmysql:scalarSync('SELECT job FROM users WHERE identifier = "'..id2..'"')

-- You can also used named parameters, which may be useful if sending the same parameter multiple times
exports.oxmysql:update('INSERT INTO ox_inventory (owner, name, data) VALUES (:owner, :name, :data) ON DUPLICATE KEY UPDATE data = :data', {
	owner = inv.owner,
	name = inv.owner,
	data = inventory,
}, function (result)
	print(('Affected %s rows'):format(result))
end)
```

## Sync vs. async
All queries are sent asychronously to the MySQL server and will await a response before sending any sort of result, without halting the execution and response of other queries.

These two variants of exports instead describe the behaviour of code being executed following a function call. Utilising a sync export will halt the execution of a function and returns inline (left-hand) results.

Conversely, the standard async exports will not interupt functions and instead return callback functions with the results being locally scoped.

```lua
-- Async
print('This print will display first')
exports.oxmysql:scalar('SELECT group FROM users WHERE identifier = ?', {playerIdentifier}, function(result)
  print('Group: '..result)
end)
print('This print will display second')

-- Sync
print('This print will display first')
local result =exports.oxmysql:scalarSync('SELECT group FROM users WHERE identifier = ?', {playerIdentifier})
print('Group: '..result)
print('This print will display third')
```

## Benchmark
Benchmark results and functions for testing the time to resolve javascript sync exports. These times are not the same as query execution time.  
Lua performance generally falls slightly behind due to overhead from cross-language communication.

Lua
`Low: 0.2955ms | High: 16.7566ms | Avg: 0.36956378ms | Total: 3695.6378ms (10000 queries)`
```lua
local lmprof = require 'lmprof'
local profiler = lmprof.create("time")
	:set_option("load_stack", true)
	:set_option("mismatch", true)
	:calibrate()

local val = 10000
RegisterCommand('luasync', function()
	local queryTimesLocal = {}
	local result
	for i=1, val do
		profiler:start()
		local r = exports.oxmysql:scalarSync('SELECT * from users')
		queryTimesLocal[#queryTimesLocal+1] = profiler:stop() / 1000000
		if i==1 then result = r end
	end
	local queryMsLow, queryMsHigh, queryMsSum = 1000, 0, 0
	for _, v in pairs(queryTimesLocal) do
		queryMsSum = queryMsSum + v
	end
	for _, v in pairs(queryTimesLocal) do
		if v > queryMsHigh then queryMsHigh = v end
	end
	for _, v in pairs(queryTimesLocal) do
		if v < queryMsLow then queryMsLow = v end
	end
	local averageQueryTime = queryMsSum / #queryTimesLocal
	print(json.encode(result))
	print('Low: '.. queryMsLow ..'ms | High: '..queryMsHigh..'ms | Avg: '..averageQueryTime..'ms | Total: '..queryMsSum..'ms ('..#queryTimesLocal..' queries)')
end)
```

Javascript
`Low: 0.2831ms | High: 5.1899ms | Avg: 0.3349578800000005ms | Total: 3349.578800000005ms (10000 queries)`
```js
const val = 10000
RegisterCommand('jssync', async() => {
    const queryTimesLocal = [];
	let result
    for(let i=0; i < val; i++) {
        const startTime = process.hrtime.bigint()
        const r = await exports.oxmysql.scalarSync('SELECT * from users')
        queryTimesLocal.push(Number(process.hrtime.bigint() - startTime) / 1000000)
        if (i === 0) result = 1
    }
    const queryMsSum = queryTimesLocal.reduce((a, b) => a + b, 0)
    const queryMsHigh = queryTimesLocal.sort((a, b) => b - a)[0]
    const queryMsLow = queryTimesLocal.sort((a, b) => a - b)[0]
    const averageQueryTime = queryMsSum / queryTimesLocal.length
	console.log(result)
    console.log('Low: '+ queryMsLow +'ms | High: '+queryMsHigh+'ms | Avg: '+averageQueryTime+'ms | Total: '+queryMsSum+'ms ('+queryTimesLocal.length+' queries)')
})
```
