

```cpp
#include <boost/python.hpp>
#include <boost/interprocess/managed_shared_memory.hpp>
#include <boost/interprocess/sync/named_mutex.hpp>
#include <Eigen/Dense>
#include <Eigen/Eigenvalues>
#include <Eigen/SVD>
#include <string>
#include <iostream>
#include <chrono>
#include <vector>

using namespace boost::python;
using namespace Eigen;
class SharedMemory {
    public:
        SharedMemory()
            : managed_shm(boost::interprocess::open_or_create, "shm", 1024),
              mutex(boost::interprocess::open_or_create, "mtx")
        {
            mutex.lock();
            int *i = managed_shm.find_or_construct<int>("Integer")();
            *i = 0;
            std::cout << "Created" << std::endl;
        }
        ~SharedMemory() {
            managed_shm.destroy<int>("Integer");
            mutex.unlock();
            std::cout << "Destroyed" << std::endl;
        }
        void increment() {
            int *i = managed_shm.find_or_construct<int>("Integer")();
            (*i)++;
            std::cout << "Incremented" << std::endl;
        }
    private:
        // 禁用复制构造和赋值
        SharedMemory(const SharedMemory&) = delete;
        SharedMemory& operator=(const SharedMemory&) = delete;
        boost::interprocess::managed_shared_memory managed_shm;
        boost::interprocess::named_mutex mutex;
};

// ==================== 辅助函数：Eigen 矩阵转 Python 列表 ====================

  
// 将 MatrixXd 转换为 Python 嵌套列表
list matrix_to_python_list(const MatrixXd& matrix) {
    list result;
    for (int i = 0; i < matrix.rows(); ++i) {
        list row;
        for (int j = 0; j < matrix.cols(); ++j) {
            row.append(matrix(i, j));
        }
        result.append(row);
    }
    return result;
}

  
// 将 VectorXd 转换为 Python 列表
list vector_to_python_list(const VectorXd& vec) {
    list result;
    for (int i = 0; i < vec.size(); ++i) {
        result.append(vec(i));
    }
    return result;
}

// ==================== Eigen3 耗时任务函数 ====================


// 大矩阵乘法 - 计算两个大矩阵的乘积（返回 Python 列表）
list matrix_multiply(int size) {
    MatrixXd A = MatrixXd::Random(size, size);
    MatrixXd B = MatrixXd::Random(size, size);
    MatrixXd C = A * B;
    return matrix_to_python_list(C);
}

  

// 特征值分解 - 计算矩阵的所有特征值和特征向量（返回 Python 列表）
list compute_eigenvalues(int size) {
    MatrixXd A = MatrixXd::Random(size, size);
    // 使矩阵对称以确保实数特征值
    A = (A + A.transpose()) / 2.0;
    SelfAdjointEigenSolver<MatrixXd> solver(size);
    solver.compute(A);
    VectorXd eigenvalues = solver.eigenvalues();
    return vector_to_python_list(eigenvalues);
}
 
// SVD 分解 - 奇异值分解（返回 Python 列表）
list compute_svd(int rows, int cols) {
    MatrixXd A = MatrixXd::Random(rows, cols);
    JacobiSVD<MatrixXd> svd(A, ComputeThinU | ComputeThinV);
    VectorXd singular_values = svd.singularValues();
    return vector_to_python_list(singular_values);
}

// 矩阵求逆 - 计算大矩阵的逆矩阵（返回 Python 列表）
list matrix_inverse(int size) {
    MatrixXd A = MatrixXd::Random(size, size);
    // 添加单位矩阵的倍数以确保矩阵可逆
    A += MatrixXd::Identity(size, size) * 0.1;
    MatrixXd inv = A.inverse();
    return matrix_to_python_list(inv);
}

// 矩阵幂运算 - 计算矩阵的 n 次幂（返回 Python 列表）
list matrix_power(int size, int power) {
    MatrixXd A = MatrixXd::Random(size, size);
    MatrixXd result = MatrixXd::Identity(size, size);
    for (int i = 0; i < power; ++i) {
        result = result * A;
    }
    return matrix_to_python_list(result);
}

// 线性方程组求解 - 求解 Ax = b（返回 Python 列表）
list solve_linear_system(int size) {
    MatrixXd A = MatrixXd::Random(size, size);
    A += MatrixXd::Identity(size, size) * 0.1; // 确保可逆
    VectorXd b = VectorXd::Random(size);
    VectorXd x = A.colPivHouseholderQr().solve(b);
    return vector_to_python_list(x);
}

// 矩阵行列式计算

double compute_determinant(int size) {
    MatrixXd A = MatrixXd::Random(size, size);
    A += MatrixXd::Identity(size, size) * 0.1;
    return A.determinant();
}

  
// 矩阵的 Cholesky 分解（用于对称正定矩阵）（返回 Python 列表）
list cholesky_decomposition(int size) {
    MatrixXd A = MatrixXd::Random(size, size);
    // 构造对称正定矩阵
    A = A * A.transpose();
    A += MatrixXd::Identity(size, size) * 0.1;
    LLT<MatrixXd> llt(A);
    MatrixXd L = llt.matrixL();
    return matrix_to_python_list(L);
}
  
// 批量矩阵运算 - 执行多次矩阵乘法（返回 Python 列表）
list batch_matrix_operations(int size, int iterations) {
    MatrixXd result = MatrixXd::Identity(size, size);
    for (int i = 0; i < iterations; ++i) {
        MatrixXd A = MatrixXd::Random(size, size);
        result = result * A;
    }
    return matrix_to_python_list(result);
}


// 带时间测量的矩阵乘法（返回执行时间，单位：毫秒）
double timed_matrix_multiply(int size) {
    auto start = std::chrono::high_resolution_clock::now();
    MatrixXd A = MatrixXd::Random(size, size);
    MatrixXd B = MatrixXd::Random(size, size);
    MatrixXd C = A * B;
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
    return duration.count() / 1000.0; // 转换为毫秒
}

  

// ==================== Python 模块导出 ====================
BOOST_PYTHON_MODULE(SharedMemoryModule) {
    // 原有的 SharedMemory 类
    class_<SharedMemory, boost::noncopyable>("SharedMemory")
        .def("increment", &SharedMemory::increment);
    // Eigen3 耗时任务函数
    def("matrix_multiply", &matrix_multiply, "Multiply two large matrices");
    def("compute_eigenvalues", &compute_eigenvalues, "Compute eigenvalues of a matrix");
    def("compute_svd", &compute_svd, "Compute SVD decomposition");
    def("matrix_inverse", &matrix_inverse, "Compute matrix inverse");
    def("matrix_power", &matrix_power, "Compute matrix power");
    def("solve_linear_system", &solve_linear_system, "Solve linear system Ax = b");
    def("compute_determinant", &compute_determinant, "Compute matrix determinant");
    def("cholesky_decomposition", &cholesky_decomposition, "Compute Cholesky decomposition");
    def("batch_matrix_operations", &batch_matrix_operations, "Perform batch matrix operations");
    def("timed_matrix_multiply", &timed_matrix_multiply, "Matrix multiplication with timing");
}
```

```cmake
cmake_minimum_required(VERSION 3.10.0)
project(WindowsApiSolutions VERSION 0.1.0 LANGUAGES C CXX)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
  
find_package(fmt CONFIG REQUIRED)
find_package(Python3 COMPONENTS Interpreter Development REQUIRED)
find_package(Boost REQUIRED COMPONENTS system filesystem thread date_time python)
find_package(Eigen3 CONFIG REQUIRED)
# Python 扩展模块是共享库
add_library(SharedMemoryModule SHARED src/main.cpp)
set_target_properties(SharedMemoryModule PROPERTIES
    PREFIX ""
    OUTPUT_NAME "SharedMemoryModule"
    # Windows 上 Python 扩展应该使用 .pyd 扩展名
    SUFFIX ".pyd"
    # 开启 DLL 聚合：自动导出所有符号（无需手动写 .def 文件）
    CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON
)
target_include_directories(SharedMemoryModule PRIVATE ${Python3_INCLUDE_DIRS})
target_link_libraries(SharedMemoryModule PRIVATE
    fmt::fmt
    Boost::boost
    Boost::system
    Boost::filesystem
    Boost::thread
    Boost::date_time
    Boost::python
    Eigen3::Eigen
    ${Python3_LIBRARIES}
)
  
include(CTest)
enable_testing()
set(CPACK_PROJECT_NAME ${PROJECT_NAME})
set(CPACK_PROJECT_VERSION ${PROJECT_VERSION})
include(CPack)
```


stubgen -m SharedMemoryModule 工具得到pyi类型存根文件。