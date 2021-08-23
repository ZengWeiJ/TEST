local blacklist = KEYS[1]; --黑名单
local ip = ARGV[1];--请求ip
local time = ARGV[2]; --过期时间
local max_count = tonumber(ARGV[3]); --最大请求数
local count = redis.call('incrby',ip,1); --请求计数器
if count == 1 then
    redis.call('expire',ip,time) --首次访问设置过期时间
elseif count > max_count then
    redis.call('sadd',blacklist,ip) --如果大于最大请求数加入到黑名单
end    
