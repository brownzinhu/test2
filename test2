local __DARKLUA_BUNDLE_MODULES

__DARKLUA_BUNDLE_MODULES = {
    cache = {},
    load = function(m)
        if not __DARKLUA_BUNDLE_MODULES.cache[m] then
            __DARKLUA_BUNDLE_MODULES.cache[m] = {
                c = __DARKLUA_BUNDLE_MODULES[m](),
            }
        end

        return __DARKLUA_BUNDLE_MODULES.cache[m].c
    end,
}

do
    function __DARKLUA_BUNDLE_MODULES.a()
        local function VIDE_ASSERT(msg)
            error(msg, 0)
        end

        return VIDE_ASSERT
    end
    function __DARKLUA_BUNDLE_MODULES.b()
        local function inline_test()
            return debug.info(1, 'n')
        end

        local is_O2 = inline_test() ~= 'inline_test'

        return {
            strict = not is_O2,
            batch = false,
        }
    end
    function __DARKLUA_BUNDLE_MODULES.c()
        local throw = __DARKLUA_BUNDLE_MODULES.load('a')
        local flags = __DARKLUA_BUNDLE_MODULES.load('b')
        local scopes = {n = 0}

        local function ycall(fn, arg)
            local thread = coroutine.create(xpcall)

            local function efn(err)
                return debug.traceback(err, 3)
            end

            local resume_ok, run_ok, result = coroutine.resume(thread, fn, efn, arg)

            assert(resume_ok)

            if coroutine.status(thread) ~= 'dead' then
                return false, debug.traceback(thread, 'attempt to yield in reactive scope')
            end

            return run_ok, result
        end
        local function get_scope()
            return scopes[scopes.n]
        end
        local function assert_stable_scope()
            local scope = get_scope()

            if not scope then
                local caller_name = debug.info(2, 'n')

                return throw(string.format('cannot use %s() outside a stable or reactive scope', tostring(caller_name)))
            elseif scope.effect then
                throw(
[[cannot create a new reactive scope inside another reactive scope]])
            end

            return scope
        end
        local function push_child(parent, child)
            table.insert(parent, child)
            table.insert(child.parents, parent)
        end
        local function push_scope(node)
            local n = scopes.n + 1

            scopes.n = n
            scopes[n] = node
        end
        local function pop_scope()
            local n = scopes.n

            scopes.n = n - 1
            scopes[n] = nil
        end
        local function push_cleanup(node, cleanup)
            if node.cleanups then
                table.insert(node.cleanups, cleanup)
            else
                node.cleanups = {cleanup}
            end
        end
        local function flush_cleanups(node)
            if node.cleanups then
                for _, fn in next, node.cleanups do
                    local ok, err = pcall(fn)

                    if not ok then
                        throw(string.format('cleanup error: %s', tostring(err)))
                    end
                end

                table.clear(node.cleanups)
            end
        end
        local function find_and_swap_pop(t, v)
            local i = (table.find(t, v))
            local n = #t

            t[i] = t[n]
            t[n] = nil
        end
        local function unparent(node)
            local parents = node.parents

            for i, parent in parents do
                find_and_swap_pop(parent, node)

                parents[i] = nil
            end
        end
        local function destroy(node)
            flush_cleanups(node)
            unparent(node)

            if node.owner then
                find_and_swap_pop((node.owner.owned), node)

                node.owner = false
            end
            if node.owned then
                local owned = node.owned

                while owned[1] do
                    destroy(owned[1])
                end
            end
        end
        local function destroy_owned(node)
            if node.owned then
                local owned = node.owned

                while owned[1] do
                    destroy(owned[1])
                end
            end
        end

        local update_queue = {n = 0}

        local function evaluate_node(node)
            if flags.strict then
                local initial_value = node.cache

                for i = 1, 2 do
                    local cur_value = node.cache

                    flush_cleanups(node)
                    destroy_owned(node)
                    push_scope(node)

                    local ok, new_value = ycall((node.effect), cur_value)

                    pop_scope()

                    if not ok then
                        table.clear(update_queue)

                        update_queue.n = 0

                        throw(string.format('effect stacktrace:\n%s', tostring(new_value)))
                    end

                    node.cache = new_value
                end

                return initial_value ~= node.cache
            else
                local cur_value = node.cache

                flush_cleanups(node)
                destroy_owned(node)
                push_scope(node)

                local ok, new_value = pcall((node.effect), node.cache)

                pop_scope()

                if not ok then
                    table.clear(update_queue)

                    update_queue.n = 0

                    throw(string.format('effect stacktrace:\n%s\n', tostring(new_value)))
                end

                node.cache = new_value

                return cur_value ~= new_value
            end
        end
        local function queue_children_for_update(node)
            local i = update_queue.n

            while node[1] do
                i = i + 1
                update_queue[i] = node[1]

                unparent(node[1])
            end

            update_queue.n = i
        end
        local function get_update_queue_length()
            return update_queue.n
        end
        local function flush_update_queue(from)
            local i = from + 1

            while i <= update_queue.n do
                local node = update_queue[i]

                if node.owner and evaluate_node(node) then
                    queue_children_for_update(node)
                end

                update_queue[i] = false
                i = i + 1
            end

            update_queue.n = from
        end
        local function update_descendants(root)
            local n0 = update_queue.n

            queue_children_for_update(root)

            if flags.batch then
                return
            end

            local i = n0 + 1

            while i <= update_queue.n do
                local node = update_queue[i]

                if node.owner and evaluate_node(node) then
                    queue_children_for_update(node)
                end

                update_queue[i] = false
                i = i + 1
            end

            update_queue.n = n0
        end
        local function push_child_to_scope(node)
            local scope = get_scope()

            if scope and scope.effect then
                push_child(node, scope)
            end
        end
        local function create_node(owner, effect, value)
            local node = {
                cache = value,
                effect = effect,
                cleanups = false,
                context = false,
                owner = owner,
                owned = false,
                parents = {},
            }

            if owner then
                if owner.owned then
                    table.insert(owner.owned, node)
                else
                    owner.owned = {node}
                end
            end

            return node
        end
        local function create_source_node(value)
            return {cache = value}
        end
        local function get_children(node)
            return {
                unpack(node),
            }
        end
        local function set_context(node, key, value)
            if node.context then
                node.context[key] = value
            else
                node.context = {[key] = value}
            end
        end

        return table.freeze{
            push_scope = push_scope,
            pop_scope = pop_scope,
            evaluate_node = evaluate_node,
            get_scope = get_scope,
            assert_stable_scope = assert_stable_scope,
            push_cleanup = push_cleanup,
            destroy = destroy,
            flush_cleanups = flush_cleanups,
            push_child_to_scope = push_child_to_scope,
            update_descendants = update_descendants,
            push_child = push_child,
            create_node = create_node,
            create_source_node = create_source_node,
            get_children = get_children,
            flush_update_queue = flush_update_queue,
            get_update_queue_length = get_update_queue_length,
            set_context = set_context,
            scopes = scopes,
        }
    end
    function __DARKLUA_BUNDLE_MODULES.d()
        local throw = __DARKLUA_BUNDLE_MODULES.load('a')
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')
        local create_node = graph.create_node
        local push_scope = graph.push_scope
        local pop_scope = graph.pop_scope
        local destroy = graph.destroy
        local refs = {}

        local function root(fn)
            local node = create_node(false, false, false)

            refs[node] = true

            local destroy = function()
                if not refs[node] then
                    throw'root already destroyed'
                end

                refs[node] = nil

                destroy(node)
            end

            push_scope(node)

            local function efn(err)
                return debug.traceback(err, 3)
            end

            local result = {
                xpcall(fn, efn, destroy),
            }

            pop_scope()

            if not result[1] then
                destroy()
                throw(string.format('error while running root():\n\n%s', tostring(result[2])))
            end

            return destroy, unpack(result, 2)
        end

        return root
    end
    function __DARKLUA_BUNDLE_MODULES.e()
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')
        local create_node = graph.create_node
        local assert_stable_scope = graph.assert_stable_scope
        local evaluate_node = graph.evaluate_node

        function create_implicit_effect(updater, binding)
            evaluate_node(create_node(assert_stable_scope(), updater, binding))
        end

        local function update_property_effect(p)
            ((p.instance))[p.property] = p.source()

            return p
        end
        local function update_parent_effect(p)
            p.instance.Parent = p.parent()

            return p
        end
        local function update_children_effect(p)
            local cur_children_set = p.cur_children_set
            local new_child_set = p.new_children_set
            local new_children = p.children()

            if type(new_children) ~= 'table' then
                new_children = {new_children}
            end

            local function process_child(child)
                if type(child) == 'table' then
                    for _, child in next, child do
                        process_child(child)
                    end
                else
                    if new_child_set[child] then
                        return
                    end

                    new_child_set[child] = true

                    if not cur_children_set[child] then
                        child.Parent = p.instance
                    else
                        cur_children_set[child] = nil
                    end
                end
            end

            process_child(new_children)

            for child in next, cur_children_set do
                child.Parent = nil
            end

            table.clear(cur_children_set)

            p.cur_children_set, p.new_children_set = new_child_set, cur_children_set

            return p
        end

        return {
            property = function(instance, property, source)
                return create_implicit_effect(update_property_effect, {
                    instance = instance,
                    property = property,
                    source = source,
                })
            end,
            parent = function(instance, parent)
                return create_implicit_effect(update_parent_effect, {
                    instance = instance,
                    parent = parent,
                })
            end,
            children = function(instance, children)
                return create_implicit_effect(update_children_effect, {
                    instance = instance,
                    cur_children_set = {},
                    new_children_set = {},
                    children = children,
                })
            end,
        }
    end
    function __DARKLUA_BUNDLE_MODULES.f()
        local ActionMT = table.freeze{}

        local function is_action(v)
            return getmetatable(v) == ActionMT
        end
        local function action(callback, priority)
            local a = {
                priority = priority or 1,
                callback = callback,
            }

            setmetatable(a, ActionMT)

            return table.freeze(a)
        end

        return function()
            return action, is_action
        end
    end
    function __DARKLUA_BUNDLE_MODULES.g()
        local flags = __DARKLUA_BUNDLE_MODULES.load('b')
        local throw = __DARKLUA_BUNDLE_MODULES.load('a')
        local bind = __DARKLUA_BUNDLE_MODULES.load('e')
        local _, is_action = __DARKLUA_BUNDLE_MODULES.load('f')()
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')
        local free_caches

        local function borrow_caches()
            if free_caches then
                local caches = free_caches

                free_caches = nil

                return caches
            else
                return {
                    events = {},
                    actions = setmetatable({}, {
                        __index = function(self, i)
                            self[i] = {}

                            return self[i]
                        end,
                    }),
                    nested_debug = setmetatable({}, {
                        __index = function(self, i)
                            self[i] = {}

                            return self[i]
                        end,
                    }),
                    nested_stack = {},
                }
            end
        end
        local function return_caches(caches)
            free_caches = caches
        end

        local aggregates = {}

        for name, class in {
            CFrame = CFrame,
            Color3 = Color3,
            UDim = UDim,
            UDim2 = UDim2,
            Vector2 = Vector2,
            Vector3 = Vector3,
            Rect = Rect,
        }do
            aggregates[name] = class.new
        end

        local function apply(instance, properties)
            if not properties then
                throw(
[[attempt to call a constructor returned by create() with no properties]])
            end

            local strict = flags.strict
            local parent = properties.Parent
            local caches = borrow_caches()
            local events = caches.events
            local actions = caches.actions
            local nested_debug = caches.nested_debug
            local nested_stack = caches.nested_stack
            local depth = 1

            repeat
                for property, value in properties do
                    local __DARKLUA_CONTINUE_13 = false

                    repeat
                        if property == 'Parent' then
                            __DARKLUA_CONTINUE_13 = true

                            break
                        end
                        if type(property) == 'string' then
                            if strict then
                                if nested_debug[depth][property] then
                                    throw(string.format('duplicate property %s at depth %s', tostring(property), tostring(depth)))
                                end

                                nested_debug[depth][property] = true
                            end
                            if type(value) == 'table' then
                                local ctor = aggregates[typeof((instance)[property])]

                                if ctor == nil then
                                    throw(string.format('cannot aggregate type %s for property %s', tostring(typeof(value)), tostring(property)))
                                end

                                (instance)[property] = ctor(unpack(value))
                            elseif type(value) == 'function' then
                                if typeof((instance)[property]) == 'RBXScriptSignal' then
                                    events[property] = value
                                else
                                    bind.property(instance, property, value)
                                end
                            else
                                (instance)[property] = value
                            end
                        elseif type(property) == 'number' then
                            if type(value) == 'function' then
                                bind.children(instance, value)
                            elseif type(value) == 'table' then
                                if is_action(value) then
                                    table.insert(actions[(value).priority], ((value).callback))
                                else
                                    table.insert(nested_stack, value)
                                    table.insert(nested_stack, depth + 1)
                                end
                            else
                                (value).Parent = instance
                            end
                        end

                        __DARKLUA_CONTINUE_13 = true
                    until true

                    if not __DARKLUA_CONTINUE_13 then
                        break
                    end
                end

                depth = (table.remove(nested_stack))
                properties = (table.remove(nested_stack))
            until not properties

            for event, listener in next, events do
                (instance)[event]:Connect(listener)
            end
            for _, queued in next, actions do
                for _, callback in next, queued do
                    callback(instance)
                end
            end

            if parent then
                if type(parent) == 'function' then
                    bind.parent(instance, parent)
                else
                    instance.Parent = parent
                end
            end

            table.clear(events)

            for _, queued in next, actions do
                table.clear(queued)
            end

            if strict then
                table.clear(nested_debug)
            end

            table.clear(nested_stack)
            return_caches(caches)

            return instance
        end

        return apply
    end
    function __DARKLUA_BUNDLE_MODULES.h()
        local root = __DARKLUA_BUNDLE_MODULES.load('d')
        local apply = __DARKLUA_BUNDLE_MODULES.load('g')

        local function mount(component, target)
            return root(function()
                local result = component()

                if target then
                    apply(target, {result})
                end
            end)
        end

        return mount
    end
    function __DARKLUA_BUNDLE_MODULES.i()
        return {
            Part = {
                Material = Enum.Material.SmoothPlastic,
                Size = Vector3.new(1, 1, 1),
                Anchored = true,
            },
            BillboardGui = {
                ResetOnSpawn = false,
                ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
            },
            CanvasGroup = nil,
            Frame = {
                BackgroundColor3 = Color3.new(1, 1, 1),
                BorderColor3 = Color3.new(0, 0, 0),
                BorderSizePixel = 0,
            },
            ImageButton = {
                BackgroundColor3 = Color3.new(1, 1, 1),
                BorderColor3 = Color3.new(0, 0, 0),
                BorderSizePixel = 0,
                AutoButtonColor = false,
            },
            ImageLabel = {
                BackgroundColor3 = Color3.new(1, 1, 1),
                BorderColor3 = Color3.new(0, 0, 0),
                BorderSizePixel = 0,
            },
            ScreenGui = {
                ResetOnSpawn = false,
                ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
            },
            ScrollingFrame = {
                BackgroundColor3 = Color3.new(1, 1, 1),
                BorderColor3 = Color3.new(0, 0, 0),
                BorderSizePixel = 0,
                ScrollBarImageColor3 = Color3.new(0, 0, 0),
            },
            SurfaceGui = {
                ResetOnSpawn = false,
                ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
                PixelsPerStud = 50,
                SizingMode = Enum.SurfaceGuiSizingMode.PixelsPerStud,
            },
            TextBox = {
                BackgroundColor3 = Color3.new(1, 1, 1),
                BorderColor3 = Color3.new(0, 0, 0),
                BorderSizePixel = 0,
                ClearTextOnFocus = false,
                Font = Enum.Font.SourceSans,
                Text = '',
                TextColor3 = Color3.new(0, 0, 0),
            },
            TextButton = {
                BackgroundColor3 = Color3.new(1, 1, 1),
                BorderColor3 = Color3.new(0, 0, 0),
                BorderSizePixel = 0,
                AutoButtonColor = false,
                Font = Enum.Font.SourceSans,
                Text = '',
                TextColor3 = Color3.new(0, 0, 0),
            },
            TextLabel = {
                BackgroundColor3 = Color3.new(1, 1, 1),
                BorderColor3 = Color3.new(0, 0, 0),
                BorderSizePixel = 0,
                Font = Enum.Font.SourceSans,
                Text = '',
                TextColor3 = Color3.new(0, 0, 0),
            },
            UIListLayout = {
                SortOrder = Enum.SortOrder.LayoutOrder,
            },
            UIGridLayout = {
                SortOrder = Enum.SortOrder.LayoutOrder,
            },
            UITableLayout = {
                SortOrder = Enum.SortOrder.LayoutOrder,
            },
            UIPageLayout = {
                SortOrder = Enum.SortOrder.LayoutOrder,
            },
            VideoFrame = {
                BackgroundColor3 = Color3.new(1, 1, 1),
                BorderColor3 = Color3.new(0, 0, 0),
                BorderSizePixel = 0,
            },
            ViewportFrame = {
                BackgroundColor3 = Color3.new(1, 1, 1),
                BorderColor3 = Color3.new(0, 0, 0),
                BorderSizePixel = 0,
            },
        }
    end
    function __DARKLUA_BUNDLE_MODULES.j()
        local throw = __DARKLUA_BUNDLE_MODULES.load('a')
        local defaults = __DARKLUA_BUNDLE_MODULES.load('i')
        local apply = __DARKLUA_BUNDLE_MODULES.load('g')
        local ctor_cache = {}

        setmetatable(ctor_cache, {
            __index = function(self, class)
                local ok, instance = pcall(Instance.new, class)

                if not ok then
                    throw(string.format('invalid class name, could not create instance of class %s', tostring(class)))
                end

                local default = defaults[class]

                if default then
                    for i, v in next, default do
                        (instance)[i] = v
                    end
                end

                local function ctor(properties)
                    return apply(instance:Clone(), properties)
                end

                self[class] = ctor

                return ctor
            end,
        })

        local function create_instance(class)
            return ctor_cache[class]
        end
        local function clone_instance(instance)
            return function(properties)
                local clone = instance:Clone()

                if not clone then
                    throw'attempt to clone a non-archivable instance'
                end

                return apply(clone, properties)
            end
        end
        local function create(class_or_instance)
            if type(class_or_instance) == 'string' then
                return create_instance(class_or_instance)
            elseif typeof(class_or_instance) == 'Instance' then
                return clone_instance(class_or_instance)
            else
                throw('bad argument #1, expected string or instance, got ' .. typeof(class_or_instance))

                return nil
            end
        end

        return (create)
    end
    function __DARKLUA_BUNDLE_MODULES.k()
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')
        local create_source_node = graph.create_source_node
        local push_child_to_scope = graph.push_child_to_scope
        local update_descendants = graph.update_descendants

        local function source(initial_value)
            local node = create_source_node(initial_value)

            return function(...)
                if select('#', ...) == 0 then
                    push_child_to_scope(node)

                    return node.cache
                end

                local v = (...)

                if node.cache == v and (type(v) ~= 'table' or table.isfrozen(v)) then
                    return v
                end

                node.cache = v

                update_descendants(node)

                return v
            end
        end

        return source
    end
    function __DARKLUA_BUNDLE_MODULES.l()
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')
        local create_node = graph.create_node
        local assert_stable_scope = graph.assert_stable_scope
        local evaluate_node = graph.evaluate_node

        local function effect(callback, initial_value)
            local node = create_node(assert_stable_scope(), callback, initial_value)

            evaluate_node(node)
        end

        return effect
    end
    function __DARKLUA_BUNDLE_MODULES.m()
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')
        local create_node = graph.create_node
        local push_child_to_scope = graph.push_child_to_scope
        local assert_stable_scope = graph.assert_stable_scope
        local evaluate_node = graph.evaluate_node

        local function derive(source)
            local node = create_node(assert_stable_scope(), source, false)

            evaluate_node(node)

            return function()
                push_child_to_scope(node)

                return node.cache
            end
        end

        return derive
    end
    function __DARKLUA_BUNDLE_MODULES.n()
        local throw = __DARKLUA_BUNDLE_MODULES.load('a')
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')
        local get_scope = graph.get_scope
        local push_cleanup = graph.push_cleanup

        local function helper(obj)
            return typeof(obj) == 'RBXScriptConnection' and function()
                obj:Disconnect()
            end or (obj.Disconnect and function()
                obj:Disconnect()
            end or (obj.Destroy and function()
                obj:Destroy()
            end or (obj.disconnect and function()
                obj:disconnect()
            end or (obj.destroy and function()
                obj:destroy()
            end or (typeof(obj) == 'Instance' and function()
                obj:Destroy()
            end or throw('cannot cleanup given object'))))))
        end
        local function cleanup(value)
            local scope = get_scope()

            if not scope then
                throw'cannot cleanup outside a stable or reactive scope'
            end

            assert(scope)

            if type(value) == 'function' then
                push_cleanup(scope, value)
            else
                push_cleanup(scope, helper(value))
            end
        end

        return cleanup
    end
    function __DARKLUA_BUNDLE_MODULES.o()
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')
        local get_scope = graph.get_scope

        local function untrack(source)
            local scope = get_scope()

            if scope then
                local effect = scope.effect

                scope.effect = false

                local ok, result = pcall(source)

                scope.effect = effect

                if not ok then
                    error(result, 0)
                end

                return result
            else
                return source()
            end
        end

        return untrack
    end
    function __DARKLUA_BUNDLE_MODULES.p()
        local function read(value)
            return (type(value) == 'function' and {
                (value()),
            } or {value})[1]
        end

        return read
    end
    function __DARKLUA_BUNDLE_MODULES.q()
        local flags = __DARKLUA_BUNDLE_MODULES.load('b')
        local throw = __DARKLUA_BUNDLE_MODULES.load('a')
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')

        local function batch(setter)
            local already_batching = flags.batch
            local from

            if not already_batching then
                flags.batch = true
                from = graph.get_update_queue_length()
            end

            local ok, err = pcall(setter)

            if not already_batching then
                flags.batch = false

                graph.flush_update_queue(from)
            end
            if not ok then
                throw(string.format('error occured while batching updates: %s', tostring(err)))
            end
        end

        return batch
    end
    function __DARKLUA_BUNDLE_MODULES.r()
        local throw = __DARKLUA_BUNDLE_MODULES.load('a')
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')
        local create_node = graph.create_node
        local get_scope = graph.get_scope
        local push_scope = graph.push_scope
        local pop_scope = graph.pop_scope
        local set_context = graph.set_context
        local nil_symbol = newproxy()
        local count = 0

        local function context(...)
            count = count + 1

            local id = count
            local has_default = select('#', ...) > 0
            local default_value = ...

            return function(...)
                local scope = get_scope()

                if select('#', ...) == 0 then
                    while scope do
                        local __DARKLUA_CONTINUE_19 = false

                        repeat
                            local ctx = scope.context

                            if not ctx then
                                scope = scope.owner
                                __DARKLUA_CONTINUE_19 = true

                                break
                            end

                            local value = (ctx)[id]

                            if value == nil then
                                scope = scope.owner
                                __DARKLUA_CONTINUE_19 = true

                                break
                            end

                            return ((value ~= nil_symbol and {value} or {nil})[1])
                        until true

                        if not __DARKLUA_CONTINUE_19 then
                            break
                        end
                    end

                    if has_default ~= nil then
                        return default_value
                    else
                        throw(
[[attempt to get context when no context is set and no default context is set]])
                    end
                else
                    if not scope then
                        return throw('attempt to set context outside of a vide scope')
                    end

                    local value, component = ...
                    local new_scope = create_node(scope, false, false)

                    set_context(new_scope, id, (value == nil and {nil_symbol} or {value})[1])
                    push_scope(new_scope)

                    local function efn(err)
                        return debug.traceback(err, 3)
                    end

                    local ok, result = xpcall(component, efn)

                    pop_scope()

                    if not ok then
                        throw(string.format('error while running context:\n\n%s', tostring(result)))
                    end

                    return result
                end

                return nil
            end
        end

        return context
    end
    function __DARKLUA_BUNDLE_MODULES.s()
        local throw = __DARKLUA_BUNDLE_MODULES.load('a')
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')
        local create_node = graph.create_node
        local evaluate_node = graph.evaluate_node
        local push_child_to_scope = graph.push_child_to_scope
        local destroy = graph.destroy
        local assert_stable_scope = graph.assert_stable_scope
        local push_scope = graph.push_scope
        local pop_scope = graph.pop_scope

        local function switch(source)
            local owner = assert_stable_scope()

            return function(map)
                local last_scope
                local last_component

                local function update(cached)
                    local component = map[source()]

                    if component == last_component then
                        return cached
                    end

                    last_component = component

                    if last_scope then
                        destroy(last_scope)

                        last_scope = nil
                    end
                    if component == nil then
                        return nil
                    end
                    if type(component) ~= 'function' then
                        throw'map must map a value to a function'
                    end

                    local new_scope = create_node(owner, false, false)

                    last_scope = new_scope

                    push_scope(new_scope)

                    local ok, result = pcall(component)

                    pop_scope()

                    if not ok then
                        error(result, 0)
                    end

                    return result
                end

                local node = create_node(owner, update, nil)

                evaluate_node(node)

                return function()
                    push_child_to_scope(node)

                    return node.cache
                end
            end
        end

        return switch
    end
    function __DARKLUA_BUNDLE_MODULES.t()
        local switch = __DARKLUA_BUNDLE_MODULES.load('s')

        local function show(source, component, fallback)
            local function truthy()
                return not not source()
            end

            return switch(truthy){
                [true] = component,
                [false] = fallback,
            }
        end

        return show
    end
    function __DARKLUA_BUNDLE_MODULES.u()
        local throw = __DARKLUA_BUNDLE_MODULES.load('a')
        local flags = __DARKLUA_BUNDLE_MODULES.load('b')
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')
        local create_node = graph.create_node
        local create_source_node = graph.create_source_node
        local push_child_to_scope = graph.push_child_to_scope
        local update_descendants = graph.update_descendants
        local assert_stable_scope = graph.assert_stable_scope
        local push_scope = graph.push_scope
        local pop_scope = graph.pop_scope
        local evaluate_node = graph.evaluate_node
        local destroy = graph.destroy

        local function check_primitives(t)
            if not flags.strict then
                return
            end

            for _, v in next, t do
                local __DARKLUA_CONTINUE_20 = false

                repeat
                    if type(v) == 'table' or type(v) == 'userdata' or type(v) == 'function' then
                        __DARKLUA_CONTINUE_20 = true

                        break
                    end

                    throw('table source map cannot return primitives')

                    __DARKLUA_CONTINUE_20 = true
                until true

                if not __DARKLUA_CONTINUE_20 then
                    break
                end
            end
        end
        local function indexes(input, transform)
            local owner = assert_stable_scope()
            local subowner = create_node(owner, false, false)
            local input_cache = {}
            local output_cache = {}
            local input_nodes = {}
            local remove_queue = {}
            local scopes = {}

            local function update_children(data)
                for i in next, input_cache do
                    if data[i] == nil then
                        table.insert(remove_queue, i)
                    end
                end
                for _, i in next, remove_queue do
                    destroy(scopes[i])

                    input_cache[i] = nil
                    output_cache[i] = nil
                    input_nodes[i] = nil
                    scopes[i] = nil
                end

                table.clear(remove_queue)
                push_scope(subowner)

                for i, v in next, data do
                    local cv = input_cache[i]

                    if cv ~= v then
                        if cv == nil then
                            local scope = create_node(subowner, false, false)

                            scopes[i] = scope

                            local node = create_source_node(v)

                            push_scope(scope)

                            local ok, result = pcall(transform, function()
                                push_child_to_scope(node)

                                return node.cache
                            end, i)

                            pop_scope()

                            if not ok then
                                pop_scope()
                                error(result, 0)
                            end

                            input_nodes[i] = node
                            output_cache[i] = result
                        else
                            input_nodes[i].cache = v

                            update_descendants(input_nodes[i])
                        end

                        input_cache[i] = v
                    end
                end

                pop_scope()

                local output_array = table.create(#scopes)

                for _, v in next, output_cache do
                    table.insert(output_array, v)
                end

                check_primitives(output_array)

                return output_array
            end

            local node = create_node(owner, function()
                return update_children(input())
            end, false)

            evaluate_node(node)

            return function()
                push_child_to_scope(node)

                return node.cache
            end
        end
        local function values(input, transform)
            local owner = assert_stable_scope()
            local subowner = create_node(owner, false, false)
            local cur_input_cache_up = {}
            local new_input_cache_up = {}
            local output_cache = {}
            local input_nodes = {}
            local scopes = {}

            local function update_children(data)
                local cur_input_cache, new_input_cache = cur_input_cache_up, new_input_cache_up

                if flags.strict then
                    local cache = {}

                    for _, v in next, data do
                        if cache[v] ~= nil then
                            throw'duplicate table value detected'
                        end

                        cache[v] = true
                    end
                end

                push_scope(subowner)

                for i, v in next, data do
                    new_input_cache[v] = i

                    local cv = cur_input_cache[v]

                    if cv == nil then
                        local scope = create_node(subowner, false, false)

                        scopes[v] = scope

                        local node = create_source_node(i)

                        push_scope(scope)

                        local ok, result = pcall(transform, v, function()
                            push_child_to_scope(node)

                            return node.cache
                        end)

                        pop_scope()

                        if not ok then
                            pop_scope()
                            error(result, 0)
                        end

                        input_nodes[v] = node
                        output_cache[v] = result
                    else
                        if cv ~= i then
                            input_nodes[v].cache = i

                            update_descendants(input_nodes[v])
                        end

                        cur_input_cache[v] = nil
                    end
                end

                pop_scope()

                for v in next, cur_input_cache do
                    destroy(scopes[v])

                    output_cache[v] = nil
                    input_nodes[v] = nil
                    scopes[v] = nil
                end

                table.clear(cur_input_cache)

                cur_input_cache_up, new_input_cache_up = new_input_cache, cur_input_cache

                local output_array = table.create(#scopes)

                for _, v in next, output_cache do
                    table.insert(output_array, v)
                end

                check_primitives(output_array)

                return output_array
            end

            local node = create_node(owner, function()
                return update_children(input())
            end, false)

            evaluate_node(node)

            return function()
                push_child_to_scope(node)

                return node.cache
            end
        end

        return function()
            return indexes, values
        end
    end
    function __DARKLUA_BUNDLE_MODULES.v()
        local throw = __DARKLUA_BUNDLE_MODULES.load('a')
        local graph = __DARKLUA_BUNDLE_MODULES.load('c')
        local create_node = graph.create_node
        local create_source_node = graph.create_source_node
        local assert_stable_scope = graph.assert_stable_scope
        local evaluate_node = graph.evaluate_node
        local update_descendants = graph.update_descendants
        local push_child_to_scope = graph.push_child_to_scope
        local UPDATE_RATE = 120
        local TOLERANCE = 0.0001

        local function Vec3(x, y, z)
            return Vector3.new(x, y, z)
        end

        local ZERO = Vec3(0, 0, 0)
        local type_to_vec6 = {
            number = function(v)
                return Vec3(v, 0, 0), ZERO
            end,
            CFrame = function(v)
                return v.Position, Vec3(v:ToEulerAnglesXYZ())
            end,
            Color3 = function(v)
                return Vec3(v.R, v.G, v.B), ZERO
            end,
            UDim = function(v)
                return Vec3(v.Scale, v.Offset, 0), ZERO
            end,
            UDim2 = function(v)
                return Vec3(v.X.Scale, v.X.Offset, v.Y.Scale), Vec3(v.Y.Offset, 0, 0)
            end,
            Vector2 = function(v)
                return Vec3(v.X, v.Y, 0), ZERO
            end,
            Vector3 = function(v)
                return v, ZERO
            end,
            Rect = function(v)
                return Vec3(v.Min.X, v.Min.Y, v.Max.X), Vec3(v.Max.Y, 0, 0)
            end,
        }
        local vec6_to_type = {
            number = function(a, b)
                return a.X
            end,
            CFrame = function(a, b)
                return CFrame.new(a) * CFrame.fromEulerAnglesXYZ(b.X, b.Y, b.Z)
            end,
            Color3 = function(v)
                return Color3.new(math.clamp(v.X, 0, 1), math.clamp(v.Y, 0, 1), math.clamp(v.Z, 0, 1))
            end,
            UDim = function(v)
                return UDim.new(v.X, math.round(v.Y))
            end,
            UDim2 = function(a, b)
                return UDim2.new(a.X, math.round(a.Y), a.Z, math.round(b.X))
            end,
            Vector2 = function(v)
                return Vector2.new(v.X, v.Y)
            end,
            Vector3 = function(v)
                return v
            end,
            Rect = function(a, b)
                return Rect.new(a.X, a.Y, a.Z, b.X)
            end,
        }
        local invalid_type = {
            __index = function(_, t)
                throw(string.format('cannot spring type %s', tostring(t)))
            end,
        }

        setmetatable(type_to_vec6, invalid_type)
        setmetatable(vec6_to_type, invalid_type)

        local springs = {}

        setmetatable(springs, {
            __mode = 'v',
        })

        local function spring(source, period, damping_ratio)
            local owner = assert_stable_scope()
            local w_n = 2 * math.pi / (period or 1)
            local z = damping_ratio or 1
            local k = w_n ^ 2
            local c_c = 2 * w_n
            local c = z * c_c

            if c > UPDATE_RATE * 2 then
                throw(
[[spring damping too high, consider reducing damping or increasing period]])
            end

            local data = {
                k = k,
                c = c,
                x0_123 = ZERO,
                x1_123 = ZERO,
                v_123 = ZERO,
                x0_456 = ZERO,
                x1_456 = ZERO,
                v_456 = ZERO,
                source_value = false,
            }
            local output = create_source_node(false)

            local function updater_effect()
                local value = source()

                data.x1_123, data.x1_456 = type_to_vec6[typeof(value)](value)
                data.source_value = value
                springs[data] = output

                return value
            end

            local updater = create_node(owner, updater_effect, false)

            evaluate_node(updater)

            data.x0_123, data.x0_456 = data.x1_123, data.x1_456
            output.cache = data.source_value

            return function(...)
                if select('#', ...) == 0 then
                    push_child_to_scope(output)

                    return output.cache
                end

                local v = (...)

                data.x0_123, data.x0_456 = type_to_vec6[typeof(v)](v)
                data.v_123 = ZERO
                data.v_456 = ZERO
                springs[data] = output
                output.cache = v

                return v
            end
        end
        local function step_springs(dt)
            for data in next, springs do
                local k, c, x0_123, x1_123, u_123, x0_456, x1_456, u_456 = data.k, data.c, data.x0_123, data.x1_123, data.v_123, data.x0_456, data.x1_456, data.v_456
                local dx_123 = x0_123 - x1_123
                local dx_456 = x0_456 - x1_456
                local fs_123 = dx_123 * -k
                local fs_456 = dx_456 * -k
                local ff_123 = u_123 * -c
                local ff_456 = u_456 * -c
                local dv_123 = (fs_123 + ff_123) * dt
                local dv_456 = (fs_456 + ff_456) * dt
                local v_123 = u_123 + dv_123
                local v_456 = u_456 + dv_456
                local x_123 = x0_123 + v_123 * dt
                local x_456 = x0_456 + v_456 * dt

                data.x0_123, da... (235 KB restante(s))
