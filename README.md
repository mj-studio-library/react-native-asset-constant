# react-native-asset-constant
There are shell scripts for generate asset(icon, string...) constants boilerplate

There is only about icon, string currently, but I will add fonts etc... later

## Icons - Purpose

We may want to image asset like following

```tsx
import { IC_MYICON } from '../../utils/Icons';
...
<Image source={IC_MYICON} />
```

instead

```tsx
<Image source={require('../../../assets/icons/myicon.png')} />
```

So we often create util file for bridge **real asset** and **constant**

<img src=https://github.com/mym0404/react-native-asset-constant/blob/master/iconasset.jpg>


_Icons.ts_
```tsx
import BACK from '../../assets/icons/back.png';
import CLOSE from '../../assets/icons/close.png';
import MASK from '../../assets/icons/mask.png';
import MyName from '../../assets/icons/myName.svg';

export const IC_BACK = BACK;
export const IC_CLOSE = CLOSE;
export const IC_MASK = MASK;

export const SVGMyName = MyName;
```

How about automatic command generate boilerplate like above?

Let's start!

## Icons - Usage

### 1. create shell script `/script/icon_constant.sh` 

```bash
# arguments parse
ASSET_DIRECTORY=$1
ICON_FILE=$2

EXCLUDE_REGEX="@3x|@2x"

ICON_FILECOUNT=0
SVG_FILECOUNT=0

# general asset
declare -a ICON_KEY_ARRAY
declare -a ICON_CONST_ARRAY
declare -a ICON_PATH_ARRAY

# svg asset
declare -a SVG_KEY_ARRAY
declare -a SVG_CONST_ARRAY
declare -a SVG_PATH_ARRAY

for image in "$ASSET_DIRECTORY"/*
do
  # filter multiplied images
  if [[ $image =~ $EXCLUDE_REGEX ]] ; then
    continue
  fi

  # filter directory or empty file
  if [[ -d "$image" ]] || ! [[ -s "$image" ]]  ; then
    continue;
  fi

  # parse image name
  assetExtension="${image##*.}" # extension
  assetNameWithExtension="${image##*/}" # filename + extension without path (after last '/' in path)
  assetName="${assetNameWithExtension%.*}" # filename without extension

  # capitlize assetName if svg file or uppercase
  if [[ $assetExtension =~ "svg" ]] ; then
    capitlizedAssetName="$(tr '[:lower:]' '[:upper:]' <<< ${assetName:0:1})${assetName:1}"

    variableName="SVG${capitlizedAssetName}"

    SVG_KEY_ARRAY[$SVG_FILECOUNT]=$capitlizedAssetName
    SVG_CONST_ARRAY[$SVG_FILECOUNT]=$variableName
    SVG_PATH_ARRAY[$SVG_FILECOUNT]=$(realpath --relative-to="$(dirname $ICON_FILE)" $image)

    SVG_FILECOUNT=$(expr $SVG_FILECOUNT + 1)
  else
    upperCasedAssetName=$(echo "$assetName" | tr "[:lower:]" "[:upper:]")

    variableName="IC_${upperCasedAssetName}"

    ICON_KEY_ARRAY[$ICON_FILECOUNT]=$upperCasedAssetName
    ICON_CONST_ARRAY[$ICON_FILECOUNT]=$variableName
    ICON_PATH_ARRAY[$ICON_FILECOUNT]=$(realpath --relative-to="$(dirname $ICON_FILE)" $image)

    ICON_FILECOUNT=$(expr $ICON_FILECOUNT + 1)
  fi


done

rm temp_image
touch temp_image

for ((i=0; i<ICON_FILECOUNT; i++)); do
  echo "import ${ICON_KEY_ARRAY[i]} from '${ICON_PATH_ARRAY[i]}';" >> temp_image
done

for ((i=0; i<SVG_FILECOUNT; i++)); do
  echo "import ${SVG_KEY_ARRAY[i]} from '${SVG_PATH_ARRAY[i]}';" >> temp_image
done

echo "" >> temp_image

for ((i=0; i<ICON_FILECOUNT; i++)); do
  echo "export const ${ICON_CONST_ARRAY[i]} = ${ICON_KEY_ARRAY[i]};" >> temp_image
done

echo "" >> temp_image

for ((i=0; i<SVG_FILECOUNT; i++)); do
  echo "export const ${SVG_CONST_ARRAY[i]} = ${SVG_KEY_ARRAY[i]};" >> temp_image
done

cp temp_image $ICON_FILE

rm temp_image

```

### 2. write and enjoy command in your `package.json`

```json
"icon": "sh ./script/icon_constant.sh your/asset/path ./src/utils/Icons.ts",
```

## Strings - Purpose

```json
{
  "YES": "예",
  "NO": "아니오",
}
```

generates

```ts
import { getString } from '../../STRINGS';

const Str = {
  YES: getString('YES'),
  NO: getString('NO'),
};
export default Str;

```

and use like this.

<img src=https://github.com/mym0404/react-native-asset-constant/blob/master/stringasset.png>

## Strings - Usage

### 1. Prepare your json i18n file

_ko.json_
```json
{
  "YES": "예",
  "NO": "아니오",
  ...
}
```

### 2. Prepare util methods from [dooboolab example reference](https://github.com/dooboolab/hackatalk-mobile/blob/master/STRINGS.ts)

They used [react-native-localize](https://github.com/react-native-community/react-native-localize) and [i18n](https://github.com/fnando/i18n-js) libraries.
There is a `getString(key: string)` method for easy translation.

_STRINGS.ts_
```ts
import * as Localization from 'react-native-localize';

import i18n from 'i18n-js';
import ko from './assets/langs/ko.json';

const locales = Localization.getLocales();

if (Array.isArray(locales)) {
  i18n.locale = locales[0].languageTag;
}

i18n.fallbacks = true;
i18n.translations = { ko };
i18n.defaultLocale = 'ko';

export const getString = (param: string, mapObj?: object): string => {
  if (mapObj) {
    return i18n.t(param, mapObj);
  }
  return i18n.t(param);
};

```

### 3. Write command to `package.json` for run shell script.

```json
"string": "sh ./script/string_constant.sh ./assets/langs/ko.json ./src/utils/Strings.ts"
```

### 4. Copy shell script

_script/string_constant.sh_
```bash
# arguments parse
STRING_ASSET_FILE=$1
STRING_UTIL_FILE=$2

COUNT=0
declare -a KEY_ARRAY

PATTERN='".*": ".*"'


while IFS= read -r line; do
  if [[ $line =~ $PATTERN ]]; then

    withoutNextLine="${line//\n/}"

    keyName=$(echo $withoutNextLine| cut -d'"' -f 2)
    echo $keyName
    KEY_ARRAY[$COUNT]=$keyName
    COUNT=$(expr $COUNT + 1)
  fi
done < $STRING_ASSET_FILE



touch temp_string

echo "import { getString } from '../../STRINGS';" >> temp_string
echo "" >> temp_string

echo "const Str = {" >> temp_string

for ((i=0; i<COUNT; i++)); do
  echo "  ${KEY_ARRAY[i]}: getString('${KEY_ARRAY[i]}')," >> temp_string
done

echo "};" >> temp_string

echo "export default Str;" >> temp_string

cp temp_string $STRING_UTIL_FILE
rm temp_string

```

### 5. Enjoy!
