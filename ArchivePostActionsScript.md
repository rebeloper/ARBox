exec > /tmp/${PROJECT_NAME}_archive.log 2>&1

UNIVERSAL_OUTPUTFOLDER=${BUILD_DIR}/${CONFIGURATION}-universal

if [ "true" == ${ALREADYINVOKED:-false} ]
then
echo "RECURSION: Detected, stopping"
else
export ALREADYINVOKED="true"

# make sure the output directory exists
mkdir -p "${UNIVERSAL_OUTPUTFOLDER}"

echo "Building for iPhoneSimulator"
xcodebuild -workspace "${WORKSPACE_PATH}" -scheme "${TARGET_NAME}" -configuration ${CONFIGURATION} -sdk iphonesimulator -destination 'platform=iOS Simulator,name=iPhone 6' ONLY_ACTIVE_ARCH=NO ARCHS='x86_64' BUILD_DIR="${BUILD_DIR}" BUILD_ROOT="${BUILD_ROOT}" ENABLE_BITCODE=YES OTHER_CFLAGS="-fembed-bitcode" BITCODE_GENERATION_MODE=bitcode clean build

# Step 1. Copy the framework structure (from iphoneos build) to the universal folder
echo "Copying to output folder"
cp -R "${ARCHIVE_PRODUCTS_PATH}${INSTALL_PATH}/${FULL_PRODUCT_NAME}" "${UNIVERSAL_OUTPUTFOLDER}/"

# Step 2. Copy Swift modules from iphonesimulator build (if it exists) to the copied framework directory
SIMULATOR_SWIFT_MODULES_DIR="${BUILD_DIR}/${CONFIGURATION}-iphonesimulator/${TARGET_NAME}.framework/Modules/${TARGET_NAME}.swiftmodule/."
if [ -d "${SIMULATOR_SWIFT_MODULES_DIR}" ]; then
cp -R "${SIMULATOR_SWIFT_MODULES_DIR}" "${UNIVERSAL_OUTPUTFOLDER}/${TARGET_NAME}.framework/Modules/${TARGET_NAME}.swiftmodule"
fi

# Step 3. Create universal binary file using lipo and place the combined executable in the copied framework directory
echo "Combining executables"
lipo -create -output "${UNIVERSAL_OUTPUTFOLDER}/${EXECUTABLE_PATH}" "${BUILD_DIR}/${CONFIGURATION}-iphonesimulator/${EXECUTABLE_PATH}" "${ARCHIVE_PRODUCTS_PATH}${INSTALL_PATH}/${EXECUTABLE_PATH}"

# Step 4. Create universal binaries for embedded frameworks
#for SUB_FRAMEWORK in $( ls "${UNIVERSAL_OUTPUTFOLDER}/${TARGET_NAME}.framework/Frameworks" ); do
#BINARY_NAME="${SUB_FRAMEWORK%.*}"
#lipo -create -output "${UNIVERSAL_OUTPUTFOLDER}/${TARGET_NAME}.framework/Frameworks/${SUB_FRAMEWORK}/${BINARY_NAME}" "${BUILD_DIR}/${CONFIGURATION}-iphonesimulator/${SUB_FRAMEWORK}/${BINARY_NAME}" "${ARCHIVE_PRODUCTS_PATH}${INSTALL_PATH}/${TARGET_NAME}.framework/Frameworks/${SUB_FRAMEWORK}/${BINARY_NAME}"
#done

# Step 5. Convenience step to copy the framework to the project's directory
echo "Copying to project dir"
yes | cp -Rf "${UNIVERSAL_OUTPUTFOLDER}/${FULL_PRODUCT_NAME}" "${PROJECT_DIR}"

#open "${PROJECT_DIR}"

# Step 6. Create zip
#cd ${PROJECT_DIR}

#mkdir Build
#cp LICENSE Build
#cp -rf ${FULL_PRODUCT_NAME} Build

#rm -rf "${FULL_PRODUCT_NAME}.zip"
#zip -r "${FULL_PRODUCT_NAME}.zip" Build
#rm -rf ${FULL_PRODUCT_NAME} Build

cd ${PROJECT_DIR}

mkdir releases
cd releases

PROJECT_VERSION=`/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" "${PROJECT_DIR}/${INFOPLIST_FILE}"`

PROJECT_VERSION_STRING=`echo $PROJECT_VERSION | awk -F "." '{print $1 "_" $2 "_" $3 }'`

rm -rf "$PROJECT_VERSION_STRING"
mkdir -p "$PROJECT_VERSION_STRING"

cd ${PROJECT_DIR}

mkdir build
cp LICENSE build
cp -rf ${FULL_PRODUCT_NAME} build

rm -rf "${PRODUCT_NAME}.zip"
zip -r "${PRODUCT_NAME}.zip" build
rm -rf ${FULL_PRODUCT_NAME} build

cp -rf "${PRODUCT_NAME}.zip" releases/"$PROJECT_VERSION_STRING"
rm -rf "${PRODUCT_NAME}.zip"

open "${PROJECT_DIR}"

fi
