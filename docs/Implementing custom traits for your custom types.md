# Implementing custom traits for your custom types
Creating custom data types in substrate requires the implementation of multiple traits, in order to ensure compatibility with the various functionalities. Some of these traits include `Clone`, `Encode`, `Decode`, `PartialEq` etc. In this Substrate-In-Bits content, we take an error-based approach to understanding why these implementations are needed and how to implement the necessary traits for your custome data types.

## Reproducing errors

### Environment and project setup
To follow along with this tutorial, ensure that you have the Rust toolchain installed.

- Visit the substrate official documentation page for the installation processes.
- Clone the project repository.
git clone https://github.com/abdbee/


- Navigate into the project’s directory.
cd 


Run the command below to compile the node.
cargo build --release


While attempting to compile the node above, you’ll encounter an error similar to the one below: