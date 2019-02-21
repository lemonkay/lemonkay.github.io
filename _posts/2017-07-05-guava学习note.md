---
    layout: post
    title: guava学习note
---



{{ page.title }}
===============


#### Joiner 06-28
 
 
 1.用设置的text代替null，重写本方法防止重复调用

``` java
 public Joiner useForNull(final String nullText) {

    checkNotNull(nullText);
    return new Joiner(this) {
      @Override
      CharSequence toString(@Nullable Object part) {
        return (part == null) ? nullText : Joiner.this.toString(part);
      }

      @Override
      public Joiner useForNull(String nullText) {
        throw new UnsupportedOperationException("already specified useForNull");
      }

      @Override
      public Joiner skipNulls() {
        throw new UnsupportedOperationException("already specified useForNull");
      }
    };
  }
``` 
2.重写size和get方法，将一个数组和两个object拼接成了一个Iterable，避免了Iterable的创建

```java
private static Iterable<Object> iterable(
      final Object first, final Object second, final Object[] rest) {
    checkNotNull(rest);
    return new AbstractList<Object>() {
      @Override
      public int size() {
        return rest.length + 2;
      }

      @Override
      public Object get(int index) {
        switch (index) {
          case 0:
            return first;
          case 1:
            return second;
          default:
            return rest[index - 2];
        }
      }
    };
  }   
``` 

#### spliter  06-29
1. 策略模式的使用

2. **continue and break label** ：到label出继续循环 或者是跳出label处的循环

```java
public static Splitter on(final String separator) {
    checkArgument(separator.length() != 0, "The separator may not be the empty string.");
    if (separator.length() == 1) {
      return Splitter.on(separator.charAt(0));
    }
    return new Splitter(
        new Strategy() {
          @Override
          public SplittingIterator iterator(Splitter splitter, CharSequence toSplit) {
            return new SplittingIterator(splitter, toSplit) {
              @Override
              public int separatorStart(int start) {
                int separatorLength = separator.length();

                positions:
                for (int p = start, last = toSplit.length() - separatorLength; p <= last; p++) {
                  for (int i = 0; i < separatorLength; i++) {
                    if (toSplit.charAt(i + p) != separator.charAt(i)) {
                      continue positions;
                    }
                  }
                  return p;
                }
                return -1;
              }

              @Override
              public int separatorEnd(int separatorPosition) {
                return separatorPosition + separator.length();
              }
            };
          }
        });
   }


```
 
   
