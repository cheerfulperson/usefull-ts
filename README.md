# Usefull ts functions

### 1. convert Camel To Snake Case

```
type CamelToSnakeCase<S extends string | number | symbol> = S extends `${infer T}${infer U}`
  ? `${T extends '1' | '2' | '3' ? '' : T extends Capitalize<T> ? '_' : ''}${Lowercase<T>}${CamelToSnakeCase<U>}`
  : S;

type TResults<T extends object> = {
  [Key in CamelToSnakeCase<keyof T>]: any;
};

export const convertCamelToSnakeCase = <T extends object>(obj: T): TResults<T> => {
  return Object.fromEntries(
    Object.entries(obj).map(([key, value]) => {
      const newKey = key.replace(/([A-Z])/g, '_$1').toLowerCase();
      return [newKey, value];
    })
  ) as TResults<T>;
};

```
