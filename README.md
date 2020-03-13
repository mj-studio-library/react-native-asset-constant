# react-native-asset-constant
There are shell scripts for generate asset(icon, string...) constants boilerplate

There is only about icon currently, but I will add string, fonts etc... later

## Icons - Needs

We may want to image asset like following

```tsx
<Image source={IC_MYICON} />
...
import { IC_MYICON } from '../../utils/Icons';
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

rm -rf temp_image

```

### 2. write and enjoy command in your `package.json`

```json
"icon": "sh ./script/image_generate.sh ./assets/icons ./src/utils/Icons.ts",
```
