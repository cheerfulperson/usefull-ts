# Usefull ts functions

### 1. Convert Camel To Snake Case

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

### 2. Earth area calculator
When you want to calculate Earth area by two poins with latitude and longitude
```
const radians = (degrees: number) => {
  return degrees * (Math.PI / 180);
};

export const getAreaRange = ({ lat, lng, distance }: { lat: number; lng: number;  distance: number /** km */ }) => {
  const lngStart = lng - distance / Math.abs(Math.cos(radians(lat)) * 111.045);
  const lngEnd = lng + distance / Math.abs(Math.cos(radians(lat)) * 111.045);
  const latStart = lat - distance / 111.045;
  const latEnd = lat + distance / 111.045;

  return {
    start: {
      lat: latStart,
      lng: lngStart,
    },
    end: {
      lat: latEnd,
      lng: lngEnd,
    },
  };
};
```

### 3. isEqual
If you need to compare several objects
```
interface IIsEqualOptions {
  strict?: boolean;
}

const isObject = (object: any): boolean => {
  return object != null && typeof object === 'object';
};

export const isEqual = <T extends object, U extends object>(
  object1: T,
  object2: U,
  options?: IIsEqualOptions,
): boolean => {
  const { strict } = options || {};
  let objKeys1 = Object.keys(object1);
  let objKeys2 = Object.keys(object2);

  if (!strict) {
    objKeys1 = objKeys1.filter(key => object1[key as keyof T] !== undefined);
    objKeys2 = objKeys2.filter(key => object2[key as keyof U] !== undefined);
  }

  if (objKeys1.length !== objKeys2.length) return false;

  for (var key of objKeys1) {
    const value1 = object1[key as keyof T] as any;
    const value2 = object2[key as keyof U] as any;

    const isObjects = isObject(value1) && isObject(value2);

    if ((isObjects && !isEqual(value1, value2)) || (!isObjects && value1 !== value2)) {
      return false;
    }
  }
  return true;
};
```

### Convert types

```
type ReplaceDecimalTypes<Type extends object, NewKeyType> = {
  [Key in keyof Type]: Type[Key] extends object[]
    ? ReplaceDecimalTypes<Type[Key][0], number>[]
    : Type[Key] extends Decimal ? NewKeyType
    : Type[Key]
}
```
