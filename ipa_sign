#!/usr/bin/env bash 

# https://github.com/badoo/ota-tools 

printUsage() {
    echo "
    Usage: $(basename "$0") Application.ipa -p foo/bar.mobileprovision -c \"iPhone Distribution: Your Company\" [-o foo/output.ipa] [-i]
    Usage: $(basename "$0") -l Application.ipa

    Options:
      -p    Provisioning profile filename               (Required)
      -c    Certficate name                             (Required)
      -o    Output filename                             (Optional)
      -i    Only inspect the package. Do not resign it. (Optional)
      
                        OR

      -l    List certificates and exit
    " 
}

# Just list certs?
if [[ "$1" == '-l' ]]; then
    security find-certificate -a | awk '/^keychain/ {if(k!=$0){print; k=$0;}} /"labl"<blob>=/{sub(".*<blob>=","          "); print}'
    exit $?
fi

# Process args                    
INSPECT_ONLY=false
IPA_IN="$1"
OPTIND=2
while getopts ``io:p:c:'' OPTION
do                                
    case $OPTION in
        i)  INSPECT_ONLY=true 
            ;;
        o)  IPA_OUT="$OPTARG"
            ;;
        p)  PROV_PROFILE="$OPTARG"
            ;;
        c)  CERT_NAME="$OPTARG"
            ;;
        ?)
            printUsage
            exit 1
            ;;
    esac                          
done                    

# Verify args
if [[ ! ( # any of the following are not true
        # IPA_IN arg is an existing regular file
        -f "$IPA_IN" &&
        # ...and it has a .ipa extension
        "${IPA_IN##*.}" == "ipa" &&
        # provisioning profile is an existing regular file
        ($INSPECT_ONLY == 1 || -f "$PROV_PROFILE") &&
        # ...and it has an .mobileprovision extension
        ($INSPECT_ONLY == 1 || "${PROV_PROFILE##*.}" == "mobileprovision") &&
        # cert name arg is a non-empty string
        ($INSPECT_ONLY == 1 || -n "$CERT_NAME")
        ) ]];
    then
        printUsage
        exit 1;
fi

# Default output file
if [ -z "$IPA_OUT" ]; then
    IPA_OUT="$(pwd)/$(basename $IPA_IN .ipa).resigned.ipa"
fi

## Exit on use of an uninitialized variable
set -o nounset
## Exit if any statement returns a non-true return value (non-zero)
set -o errexit
## Announce commands
#set -o xtrace

TMP="$(mktemp -d /tmp/resign.$(basename $IPA_IN .ipa).XXXXX)"
pushd "$TMP" > /dev/null
unzip -q "$IPA_IN"
echo "App has BundleIdentifier '$(/usr/libexec/PlistBuddy -c 'Print :CFBundleIdentifier' Payload/*.app/Info.plist)' and BundleVersion $(/usr/libexec/PlistBuddy -c 'Print :CFBundleVersion' Payload/*.app/Info.plist)"
security cms -D -i Payload/*.app/embedded.mobileprovision > mobileprovision.plist
echo "App has provision   '$(/usr/libexec/PlistBuddy -c "Print :Name" mobileprovision.plist)', which supports '$(/usr/libexec/PlistBuddy -c "Print :Entitlements:application-identifier" mobileprovision.plist)'"
if [[ ! ($INSPECT_ONLY == 1) ]]; then
    security cms -D -i "$PROV_PROFILE" > provision.plist
    echo "Embedding provision '$(/usr/libexec/PlistBuddy -c "Print :Name" provision.plist)', which supports '$(/usr/libexec/PlistBuddy -c "Print :Entitlements:application-identifier" provision.plist)'"
    rm -rf Payload/*.app/_CodeSignature Payload/*.app/CodeResources
    cp "$PROV_PROFILE" Payload/*.app/embedded.mobileprovision
    /usr/bin/codesign -f -s "$CERT_NAME" --resource-rules Payload/*.app/ResourceRules.plist Payload/*.app
    zip -qr "$IPA_OUT" Payload
fi
popd > /dev/null
rm -rf "$TMP"
