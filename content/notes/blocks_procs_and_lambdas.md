+++
date = '2025-07-03'
draft = false
title = 'Blocks, procs, and lambdas in Ruby'
+++
## Block Behavior:
```ruby

def method_that_takes_a_block
  yield
  puts "this happened in the method"
end
method_that_takes_a_block do
  puts "this happened in the block"
end
# => this happened in the block
# => this happened in the method

# yield should be protected by block_given?
# in case the caller does not provide a block
method_that_takes_a_block
# => no block given (yield) (LocalJumpError)
def method_that_takes_a_block
  if block_given?
    yield
  else
    puts "No block provided!"
  end
end
method_that_takes_a_block
# => No block provided!
```

# Proc Behavior:
```ruby
# defining a proc:
p = Proc.new { puts "I'm a proc!" }
# or this:
p = proc { puts "I'm a proc!" }

# calling a proc
p.call
# => I'm a proc!

# procs don't throw an error if passed arguments 
# don't match method signature
p.call(1, 2, 3)
# => I'm a proc!
p = Proc.new { |name| puts "Hello, #{name}" }
p.call
# => Hello,
p.call("Jake")
# => Hello, Jake

# In a proc 'return' returns from it's calling context
p = Proc.new { return nil }
p.call
# unexpected return (LocalJumpError)
def proc_in_a_method
  p = proc { return "returning from the proc" }
  p.call
  return "returning from the method"
end
puts proc_in_a_method
# => returning from the proc
```
# Lambda behavior:
```ruby
# similar to procs, but defined using 'lambda' or '->'
l = lambda { puts "I'm a lambda" }
l.call
# => I'm a lambda
l = -> { puts "Different way to declare a lambda" }
l.call
# => Different way to declare a lambda

# passed arguments must match method signature:
l = lambda { |name| puts "Hello, #{name}" }
l.call
# wrong number of arguments (given 0, expected 1) (ArgumentError)
l.call "Jake"
# => Hello, Jake

# In a lambda 'return' only returns from the lambda
 def returning_from_a_lambda(n)
  is_even = lambda { |n| return n % 2 == 0 }
  if is_even.call(n)
    puts "#{n} is even"
  else
    puts "#{n} is odd"
  end
end

returning_from_a_lambda 1
# => 1 is odd

returning_from_a_lambda 2
# => 2 is even
```