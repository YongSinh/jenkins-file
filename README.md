# jenkins-file
 // | tee maven-build.log
// // Now parse the log safely
// def match = sh(
//     script: "grep \"Successfully built image\" maven-build.log || true",
//     returnStdout: true
// ).trim()

// if (!match) {
//     error "❌ Failed to extract Docker image name for project: ${project}"
// }

// def imageName = (match =~ /'([^']+)'/)[0][1]
// echo "✅ Built Docker image: ${imageName}"

// // Save image name for later stages
// env.DOCKER_IMAGE = imageName