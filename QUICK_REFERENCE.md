# Quick Reference: Performance Optimization Patterns

This document provides quick-reference code snippets for common performance optimization patterns in React/Next.js applications.

---

## Table of Contents
1. [Memoization Patterns](#memoization-patterns)
2. [Optimization Techniques](#optimization-techniques)
3. [Common Pitfalls](#common-pitfalls)
4. [Before/After Examples](#beforeafter-examples)

---

## Memoization Patterns

### 1. useMemo for Expensive Calculations

```typescript
// ❌ BAD: Recalculated every render
function Component({ data }) {
  const sum = data.reduce((acc, item) => acc + item.value, 0)
  return <div>{sum}</div>
}

// ✅ GOOD: Calculated only when data changes
function Component({ data }) {
  const sum = useMemo(
    () => data.reduce((acc, item) => acc + item.value, 0),
    [data]
  )
  return <div>{sum}</div>
}
```

### 2. useMemo for Filtered/Sorted Lists

```typescript
// ❌ BAD: Filters on every render
function Component({ items, searchTerm }) {
  const filtered = items.filter(item => 
    item.name.toLowerCase().includes(searchTerm.toLowerCase())
  )
  return <List items={filtered} />
}

// ✅ GOOD: Filters only when dependencies change
function Component({ items, searchTerm }) {
  const normalizedSearch = useMemo(
    () => searchTerm.toLowerCase(),
    [searchTerm]
  )
  
  const filtered = useMemo(
    () => items.filter(item => 
      item.name.toLowerCase().includes(normalizedSearch)
    ),
    [items, normalizedSearch]
  )
  
  return <List items={filtered} />
}
```

### 3. useMemo for Static Data

```typescript
// ❌ BAD: Array recreated every render
function Component() {
  const options = [
    { label: 'Option 1', value: '1' },
    { label: 'Option 2', value: '2' },
  ]
  return <Select options={options} />
}

// ✅ GOOD: Array created once
function Component() {
  const options = useMemo(() => [
    { label: 'Option 1', value: '1' },
    { label: 'Option 2', value: '2' },
  ], [])
  
  return <Select options={options} />
}

// ✅ BETTER: Move outside component
const OPTIONS = [
  { label: 'Option 1', value: '1' },
  { label: 'Option 2', value: '2' },
]

function Component() {
  return <Select options={OPTIONS} />
}
```

### 4. useCallback for Event Handlers

```typescript
// ❌ BAD: New function every render
function Component({ onUpdate }) {
  const handleClick = () => {
    onUpdate({ status: 'updated' })
  }
  return <Button onClick={handleClick}>Update</Button>
}

// ✅ GOOD: Function memoized
function Component({ onUpdate }) {
  const handleClick = useCallback(() => {
    onUpdate({ status: 'updated' })
  }, [onUpdate])
  
  return <Button onClick={handleClick}>Update</Button>
}
```

---

## Optimization Techniques

### 1. Map-Based Lookups (O(1) instead of O(n))

```typescript
// ❌ BAD: O(n²) - find inside map
function Component({ matches, persons, items }) {
  return matches.map(match => {
    const person = persons.find(p => p.id === match.personId) // O(n)
    const item = items.find(i => i.id === match.itemId)       // O(n)
    return <Card person={person} item={item} />
  })
}

// ✅ GOOD: O(n) - Map-based lookup
function Component({ matches, persons, items }) {
  const personMap = useMemo(
    () => new Map(persons.map(p => [p.id, p])),
    [persons]
  )
  
  const itemMap = useMemo(
    () => new Map(items.map(i => [i.id, i])),
    [items]
  )
  
  return matches.map(match => {
    const person = personMap.get(match.personId) // O(1)
    const item = itemMap.get(match.itemId)       // O(1)
    return <Card person={person} item={item} key={match.id} />
  })
}
```

### 2. Single-Pass Algorithms

```typescript
// ❌ BAD: Multiple passes through data
function Component({ data }) {
  const total = data.reduce((sum, item) => sum + item.value, 0)
  const count = data.length
  const average = total / count
  const max = Math.max(...data.map(item => item.value))
  
  return <Stats total={total} count={count} average={average} max={max} />
}

// ✅ GOOD: Single pass through data
function Component({ data }) {
  const stats = useMemo(() => {
    let total = 0
    let max = -Infinity
    
    for (const item of data) {
      total += item.value
      if (item.value > max) max = item.value
    }
    
    return {
      total,
      count: data.length,
      average: total / data.length,
      max,
    }
  }, [data])
  
  return <Stats {...stats} />
}
```

### 3. Early Returns in Filters

```typescript
// ❌ BAD: Evaluates all conditions
function filterItems(items, statusFilter, searchTerm, categoryFilter) {
  return items.filter(item => {
    const matchesStatus = statusFilter === 'all' || item.status === statusFilter
    const matchesSearch = item.name.toLowerCase().includes(searchTerm.toLowerCase())
    const matchesCategory = categoryFilter === 'all' || item.category === categoryFilter
    return matchesStatus && matchesSearch && matchesCategory
  })
}

// ✅ GOOD: Early returns for failed conditions
function filterItems(items, statusFilter, searchTerm, categoryFilter) {
  const normalizedSearch = searchTerm.toLowerCase()
  
  return items.filter(item => {
    // Most restrictive filter first
    if (statusFilter !== 'all' && item.status !== statusFilter) return false
    if (categoryFilter !== 'all' && item.category !== categoryFilter) return false
    if (searchTerm && !item.name.toLowerCase().includes(normalizedSearch)) return false
    return true
  })
}
```

### 4. Conditional Rendering Optimization

```typescript
// ❌ BAD: Components rendered but hidden
function Component({ showDetails, data }) {
  return (
    <div>
      <Summary data={data} />
      <div style={{ display: showDetails ? 'block' : 'none' }}>
        <ExpensiveDetails data={data} />
      </div>
    </div>
  )
}

// ✅ GOOD: Components not rendered when hidden
function Component({ showDetails, data }) {
  return (
    <div>
      <Summary data={data} />
      {showDetails && <ExpensiveDetails data={data} />}
    </div>
  )
}
```

### 5. React.memo for Pure Components

```typescript
// ❌ BAD: Re-renders even when props unchanged
function ListItem({ item }) {
  return <div>{item.name}</div>
}

function List({ items }) {
  return items.map(item => <ListItem key={item.id} item={item} />)
}

// ✅ GOOD: Only re-renders when item changes
const ListItem = React.memo(function ListItem({ item }) {
  return <div>{item.name}</div>
})

function List({ items }) {
  return items.map(item => <ListItem key={item.id} item={item} />)
}
```

---

## Common Pitfalls

### 1. useMemo with Missing Dependencies

```typescript
// ❌ BAD: Missing dependency
function Component({ items, filter }) {
  const filtered = useMemo(
    () => items.filter(item => item.type === filter),
    [items] // Missing 'filter' dependency!
  )
  return <List items={filtered} />
}

// ✅ GOOD: All dependencies included
function Component({ items, filter }) {
  const filtered = useMemo(
    () => items.filter(item => item.type === filter),
    [items, filter]
  )
  return <List items={filtered} />
}
```

### 2. Memoizing Everything (Over-optimization)

```typescript
// ❌ BAD: Over-memoization for simple operations
function Component({ name }) {
  const uppercaseName = useMemo(() => name.toUpperCase(), [name])
  const nameLength = useMemo(() => name.length, [name])
  return <div>{uppercaseName} ({nameLength})</div>
}

// ✅ GOOD: Only memoize expensive operations
function Component({ name }) {
  return <div>{name.toUpperCase()} ({name.length})</div>
}
```

### 3. Creating New Objects in Dependency Arrays

```typescript
// ❌ BAD: New object every render
function Component({ userId }) {
  const user = useMemo(
    () => fetchUser(userId),
    [{ id: userId }] // New object each time!
  )
  return <UserCard user={user} />
}

// ✅ GOOD: Primitive values in dependencies
function Component({ userId }) {
  const user = useMemo(
    () => fetchUser(userId),
    [userId]
  )
  return <UserCard user={user} />
}
```

### 4. Inline Function in JSX

```typescript
// ❌ BAD: New function every render
function Component({ items }) {
  return (
    <div>
      {items.map(item => (
        <Button 
          key={item.id}
          onClick={() => handleClick(item.id)}
        >
          {item.name}
        </Button>
      ))}
    </div>
  )
}

// ✅ GOOD: Memoized callback or data attribute
function Component({ items }) {
  const handleClick = useCallback((id) => {
    // handle click
  }, [])
  
  return (
    <div>
      {items.map(item => (
        <Button 
          key={item.id}
          onClick={() => handleClick(item.id)}
        >
          {item.name}
        </Button>
      ))}
    </div>
  )
}
```

---

## Before/After Examples

### Example 1: Dashboard with Nested Lookups

```typescript
// ❌ BEFORE: O(n²) complexity
function Dashboard() {
  const matches = getMatches()
  const persons = getPersons()
  const items = getItems()
  
  return (
    <div>
      {matches.map(match => {
        const person = persons.find(p => p.id === match.personId)
        const item = items.find(i => i.id === match.itemId)
        
        return (
          <Card key={match.id}>
            <h3>{person?.name}</h3>
            <p>{item?.description}</p>
            <span>{match.score}</span>
          </Card>
        )
      })}
    </div>
  )
}

// ✅ AFTER: O(n) complexity
function Dashboard() {
  const matches = getMatches()
  const persons = getPersons()
  const items = getItems()
  
  const personMap = useMemo(
    () => new Map(persons.map(p => [p.id, p])),
    [persons]
  )
  
  const itemMap = useMemo(
    () => new Map(items.map(i => [i.id, i])),
    [items]
  )
  
  return (
    <div>
      {matches.map(match => {
        const person = personMap.get(match.personId)
        const item = itemMap.get(match.itemId)
        
        return (
          <Card key={match.id}>
            <h3>{person?.name}</h3>
            <p>{item?.description}</p>
            <span>{match.score}</span>
          </Card>
        )
      })}
    </div>
  )
}
```

### Example 2: Statistics Calculation

```typescript
// ❌ BEFORE: Multiple iterations
function StatsCard({ data }) {
  const total = data.reduce((sum, item) => sum + item.count, 0)
  const missing = data.filter(d => d.type === 'missing').reduce((sum, d) => sum + d.count, 0)
  const found = data.filter(d => d.type === 'found').reduce((sum, d) => sum + d.count, 0)
  const pending = data.filter(d => d.type === 'pending').reduce((sum, d) => sum + d.count, 0)
  
  return (
    <div>
      <Stat label="Total" value={total} />
      <Stat label="Missing" value={missing} />
      <Stat label="Found" value={found} />
      <Stat label="Pending" value={pending} />
    </div>
  )
}

// ✅ AFTER: Single iteration with memoization
function StatsCard({ data }) {
  const stats = useMemo(() => {
    let total = 0
    let missing = 0
    let found = 0
    let pending = 0
    
    for (const item of data) {
      total += item.count
      if (item.type === 'missing') missing += item.count
      else if (item.type === 'found') found += item.count
      else if (item.type === 'pending') pending += item.count
    }
    
    return { total, missing, found, pending }
  }, [data])
  
  return (
    <div>
      <Stat label="Total" value={stats.total} />
      <Stat label="Missing" value={stats.missing} />
      <Stat label="Found" value={stats.found} />
      <Stat label="Pending" value={stats.pending} />
    </div>
  )
}
```

### Example 3: Search with Filter

```typescript
// ❌ BEFORE: No memoization, multiple toLowerCase calls
function SearchableList({ items, searchTerm, statusFilter }) {
  const filtered = items.filter(item => {
    const matchesSearch = item.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
                         item.description.toLowerCase().includes(searchTerm.toLowerCase())
    const matchesStatus = statusFilter === 'all' || item.status === statusFilter
    return matchesSearch && matchesStatus
  })
  
  return <List items={filtered} />
}

// ✅ AFTER: Memoized with normalized search
function SearchableList({ items, searchTerm, statusFilter }) {
  const normalizedSearch = useMemo(
    () => searchTerm.toLowerCase().trim(),
    [searchTerm]
  )
  
  const filtered = useMemo(() => {
    return items.filter(item => {
      if (statusFilter !== 'all' && item.status !== statusFilter) {
        return false
      }
      
      if (!normalizedSearch) return true
      
      return item.name.toLowerCase().includes(normalizedSearch) ||
             item.description.toLowerCase().includes(normalizedSearch)
    })
  }, [items, normalizedSearch, statusFilter])
  
  return <List items={filtered} />
}
```

---

## Performance Measurement

### Quick DevTools Profiler Check

```typescript
// Add this to any component for quick profiling
useEffect(() => {
  console.time('ComponentRender')
  return () => console.timeEnd('ComponentRender')
})
```

### Measure Specific Operations

```typescript
function Component({ data }) {
  const result = useMemo(() => {
    console.time('ExpensiveOperation')
    const result = expensiveOperation(data)
    console.timeEnd('ExpensiveOperation')
    return result
  }, [data])
  
  return <div>{result}</div>
}
```

---

## Checklist for Code Review

When reviewing code for performance:

- [ ] Are expensive calculations wrapped in `useMemo`?
- [ ] Do `useMemo`/`useCallback` have correct dependencies?
- [ ] Are lookups optimized (Map instead of find)?
- [ ] Is data processed in single pass where possible?
- [ ] Are static data structures moved outside component?
- [ ] Are early returns used in filters?
- [ ] Is conditional rendering used instead of CSS hiding?
- [ ] Are components memoized with React.memo when appropriate?
- [ ] Are inline functions in JSX avoided or memoized?
- [ ] Is string manipulation (toLowerCase, trim) cached?

---

## Resources

- [React useMemo Documentation](https://react.dev/reference/react/useMemo)
- [React useCallback Documentation](https://react.dev/reference/react/useCallback)
- [React.memo Documentation](https://react.dev/reference/react/memo)
- [React DevTools Profiler Guide](https://react.dev/learn/react-developer-tools)

---

## Summary

**Key Takeaways:**
1. **Measure first** - Use React DevTools Profiler
2. **Memoize expensive operations** - Don't memoize everything
3. **Use proper data structures** - Map for lookups, Set for uniqueness
4. **Single-pass algorithms** - Reduce iterations
5. **Correct dependencies** - Include all dependencies in useMemo/useCallback
6. **Test thoroughly** - Ensure optimizations don't break functionality
